---
title: Reflective shellcode runner in PowerShell
date: 2025-06-24 +/-TTTT
categories: [REDTEAM]
tags: [powershell, shellcode, reflection, winapi]
---

## Part 1 - Understanding the mechanics

The blog still needs some fixes, as it currently resembles a personal note and lacks information that will be added soon.


### LookupFunc: finding Windows API modules and functions

We'll examine how to call Windows API functions from PowerShell, starting with displaying a message box.

First, we need a way to locate functions in DLLs. We can use technique known as dynamic lookup. We will create the .NET assembly in memory.
To perform a dynamic lookup of function addresses the operating system provides two Win32 APIs called `GetModuleHandle` and `GetProcAddress`.

`GetModuleHandle` obtains a handle to the specified DLL, which is the memory address of the DLL. To find address of specific function we need to pass the DLL handle and the desired function to `GetProcAddress` which will return the function address.

We are relaying on `GetAssemblies` to search preloaded assemblies in the PowerShell process. We will loop through all assemblies and invoke `GetTypes` to obtain methods and structures. When C# code wants to directly invoke Win32 APIs it must provide the Unsafe keyword. Any functions to be be used must be declared as static to avoid instantiation. All these methods are ments to be used internally by the .NET code. This blocks us calling them directly from PowerShell or C#. To call them indirectly we need to obtain reference to these functions but firstly we must obtain a reference to the `System.dll` assembly using the `Get-Type` method. This will allow to subsequently locate the `GetModuleHandle` and `GetProcAddress` methods inside this dll.

**Using `GetType` to obtain a reference to the `System.dll` assembly at runtime is an example of the Reflection technique.**

We can use internal `Invoke` method to call `GetModuleHandle` and obtain the base address of an unmanaged DLL. The returned value is decimal and we transfer it into hexadecimal quivalent to obtain the actual memory address in a format of `0xdeadbeef`. The same applies to `GetProcAddress`.

The `LookupFunc` helper function does this by:

- Accessing .NET's internal methods through reflection
- Finding GetProcAddress (which locates function addresses)
- Returning the memory address of our desired function

Function below locates memory addresses of function in DLL. We'll use it to find `MessageBoxA` from `user32.dll`.

```powershell
function LookupFunc {
    Param (
        $moduleName,
        $functionName
    )

    $assem = ([AppDomain]::CurrentDomain.GetAssemblies() | 
        Where-Object {
            $_.GlobalAssemblyCache -And 
            $_.Location.Split('\\')[-1].Equals('System.dll')
        }
    ).GetType('Microsoft.Win32.UnsafeNativeMethods')

    $tmp = @()
    
    $assem.GetMethods() | ForEach-Object {
        If ($_.Name -eq "GetProcAddress") {
            $tmp += $_
        }
    }
    
    return $tmp[0].Invoke( # Using the first `GetProcAddress` function.
        $null, 
        @(
            ($assem.GetMethod('GetModuleHandle')).Invoke(
                $null, 
                @($moduleName)
            ), 
            $functionName
        )
    )
}
```

Formatted one, to use less lines of code:

```powershell
function LookupFunc {
	Param ($moduleName, $functionName)
	$assem = ([AppDomain]::CurrentDomain.GetAssemblies() | Where-Object {$_.GlobalAssemblyCache -And $_.Location.Split('\\')[-1].Equals('System.dll')}).GetType('Microsoft.Win32.UnsafeNativeMethods')
	$tmp=@()
	$assem.GetMethods() | ForEach-Object {If($_.Name -eq "GetProcAddress") {$tmp+=$_}}
	return $tmp[0].Invoke($null, @(($assem.GetMethod('GetModuleHandle')).Invoke($null, @($moduleName)), $functionName))
	}
```

Later in our code, we will use it to save the result into a variable using format like so:

`$Variable = LookupFunc "nameof.dll" "WinAPIFunctionName"`


### DelegateType reflection - calling resolved functions

We create a DelegateType reflection to call the resolved function from unmanaged module address.

Key steps:

- `LookupFunc` finds `MessageBoxA` address in our example
- Creation of a new dynamic reflection assembly object
- Getting current AppDomain
- Defining dynamic assembly with executable permissions
- Creation of dynamic module object inside of assembly
- Custom type creation
- Defining constructor with MessageBoxA's signature
- Setting implementation flags
- Creating Invoke method
- Creating delegate type instance
- GetDelegateForFunctionPointer (Connecting our native function pointer to the managed delegate)
- Calling the function

This will result in a call to the function and message box will pop-up with our custom text.

```powershell
$MessageBoxA = LookupFunc "user32.dll" "MessageBoxA" # Saving resolved address to the variable

# 1. Create a dynamic assembly to host our delegate

$MyAssembly = New-Object System.Reflection.AssemblyName('ReflectedDelegate') # Creating a new reflection assembly object with name 'ReflectedDelegate'

$Domain = [AppDomain]::CurrentDomain # Retrieves the current application domain (AppDomain) in which the PowerShell session is running

$MyAssemblyBuilder = $Domain.DefineDynamicAssembly($MyAssembly, [System.Reflection.Emit.AssemblyBuilderAccess]::Run) # Setting executable permissions on assembly. Specifies that the assembly can only be executed in memory and cannot be saved to disk

# 2. Create a module within the assembly

$MyModuleBuilder = $MyAssemblyBuilder.DefineDynamicModule('InMemoryModule', $false) # Creating Module inside of assembly. Setting custom module name. Second $false argument will exclude symbol information.

# Now we can create a custom type that will become our delegate type.

# 3. Define our custom delegate type

$MyTypeBuilder = $MyModuleBuilder.DefineType('MyDelegateType', 'Class, Public, Sealed, AnsiClass, AutoClass',[System.MulticastDelegate]) # The first argument is a custom name. Second is the list of attributes for the type. We set it to class so that later it can be instantiated. Public, non-executable and use ASCII instead of Unicode. We set undocumented setting 'Sealed' so it to be interpreted automatically. Third argument, we specify type it builds on top of. 'MulticastDelegate' will allow us to create a delegate type with multiple entries which will allow to call the target API with multiple arguments.

# Now we can put function prototype inside the newly created Type '$MyTypeBuilder' and let it become our custom delegate type.

# 4. Create constructor matching MessageBoxA signature

$MyConstructorBuilder = $MyTypeBuilder.DefineConstructor(

	'RTSpecialName, HideBySig, Public', [System.Reflection.CallingConventions]::Standard, @([IntPtr], [String], [String], [int])

)
# The first argument - atributes of the constructor itself. We make it Public and require it to be referenced by both name and signature. (RTSpecialName, HideBySig and Public).
# The second arguemnt - calling convention for the constructor. This defines how arguments and return values are handles on the low level by .NET framework. (The default one in use.
# The last argument - parameter types of the constructor that will become the function prototype. Basically, the arguments types of the function we would like to invoke.


#With the constructor created, we must call it. Before that, implementation flags must be set.  Runtime - used at runtime. Managed - the code is managed. =)

$MyConstructorBuilder.SetImplementationFlags('Runtime, Managed')

# Constructor is set up. No need to modify it or add anything else. But, in order to invoke it, we must tell .NET that delegate type to be used in calling a function. Let's define a separate thing, the Invoke method via 'DefineMethod' which takes settings of '$MyTypeBuilder'.

# DefineMethod takes 4 arguments.
# Name of the method. Method attributes. NewSlot and Virtual in second arg used to indicate that the method is virtual and ensure it always gets a new slot in the vtable.
# Third arg - return type of the function, which for example 'MessageBoxA' is int.
# Fourth - array of argument types for the function we would like to call.

# 5. Define the Invoke method (actual function call)

$MyMethodBuilder = $MyTypeBuilder.DefineMethod('Invoke', 'Public, HideBySig, NewSlot, Virtual', [int], @([IntPtr], [String], [String], [int]))

# Just as with constructor, we must set the implementation flags to allow the Invoke method to be called.

$MyMethodBuilder.SetImplementationFlags('Runtime, Managed')

# And finally, we create the delegate type object by instantiating $MyTypeBuilder.

# 6. Finalize the type

$MyDelegateType = $MyTypeBuilder.CreateType()


# We suply to $MyFunction the result of GetDelegateForFunctionPointer() function.

# 7. Connect our function pointer to the delegate

$MyFunction = [System.Runtime.InteropServices.Marshal]::GetDelegateForFunctionPointer(
	$MessageBoxA,
	$MyDelegateType
)

# 8. Call MessageBoxA

$MyFunction.Invoke([IntPtr]::Zero, "Hello World", "This is My MessageBox!", 0)
```

If you read carefully, you might notice that we never call the defined constructor.

We don't explicitly call `MyConstructorBuilder` because it's used internally by .NET when we create the delegate with `GetDelegateForFunctionPointer` - the constructor defines the delegate's structure, and .NET automatically uses it when converting the function pointer to a managed delegate. The Invoke method is what we call directly, while the constructor works behind the scenes.

### Bonus - Invoking WinExec

Using the same technique, we can launch applications. we we will create a child process by spawning instance of `notepad.exe` process.

As previously, we declare and use `LookupFunc` to identify the memory addresses. This time, for `WinExec` from `KERNEL32.dll`.

To find the proper data types we can use resources below:

- [https://www.pinvoke.net/default.aspx/kernel32.winexec](https://www.pinvoke.net/default.aspx/kernel32.winexec)
- [https://learn.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-winexec](https://learn.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-winexec)
- [https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-showwindow](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-showwindow)

```powershell
function LookupFunc {

	Param ($moduleName, $functionName)

	$assem = ([AppDomain]::CurrentDomain.GetAssemblies() | Where-Object {$_.GlobalAssemblyCache -And $_.Location.Split('\\')[-1].Equals('System.dll')}).GetType('Microsoft.Win32.UnsafeNativeMethods')
	$tmp=@()
	$assem.GetMethods() | ForEach-Object {If($_.Name -eq "GetProcAddress") {$tmp+=$_}}
	
	return $tmp[0].Invoke($null, @(($assem.GetMethod('GetModuleHandle')).Invoke($null, @($moduleName)), $functionName))
	}

$WinExecFunc = LookupFunc "kernel32.dll" "WinExec" # Saving resolved address to the variable


$MyAssembly = New-Object System.Reflection.AssemblyName('ReflectedDelegate')

$Domain = [AppDomain]::CurrentDomain

$MyAssemblyBuilder = $Domain.DefineDynamicAssembly($MyAssembly, [System.Reflection.Emit.AssemblyBuilderAccess]::Run)

$MyModuleBuilder = $MyAssemblyBuilder.DefineDynamicModule('InMemoryModule', $false)

$MyTypeBuilder = $MyModuleBuilder.DefineType('MyDelegateType', 'Class, Public, Sealed, AnsiClass, AutoClass',[System.MulticastDelegate])

$MyConstructorBuilder = $MyTypeBuilder.DefineConstructor(

	'RTSpecialName, HideBySig, Public', [System.Reflection.CallingConventions]::Standard, @([String],[int]) # The third parameter - the arguments types of the WinExec function we would like to invoke.

)

$MyConstructorBuilder.SetImplementationFlags('Runtime, Managed')

$MyMethodBuilder = $MyTypeBuilder.DefineMethod('Invoke', 'Public, HideBySig, NewSlot, Virtual', [int], @([String],[int])) # Fourth argumtnt - array of argument types for the WinExec function.

$MyMethodBuilder.SetImplementationFlags('Runtime, Managed')

$MyDelegateType = $MyTypeBuilder.CreateType()

$MyFunction = [System.Runtime.InteropServices.Marshal]::GetDelegateForFunctionPointer($WinExecFunc, $MyDelegateType)

# Calling the function

$MyFunction.Invoke("notepad.exe", 5)
```

Key Differences:
- We use WinExec instead of MessageBoxA
- The delegate type matches WinExec's simpler signature (just String and int parameters)
- The second parameter 5 makes the window visible (SW_SHOW)

---

## Part 2 - Loading shellcode

TO BE ADDED...