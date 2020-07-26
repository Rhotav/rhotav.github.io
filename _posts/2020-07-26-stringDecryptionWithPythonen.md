---
layout: post
author: Rhotav
title: String Deobfuscation with Invoke ~ Python [EN]
---

Greetings, in this article, 
I explained how we can deobfuscate the "Crypto Obfuscator" string protection in Python using dnlib and Reflection.

If you need to ask questions contact to me from twitter or comments
(sorry for my bad english I try to improve myself.)

## File Analysis

Let's open the file we encrypted with trial version crypto obfuscator on dnSpy

When we go to a random event, we will see encrypted strings and junk codes:
![Screenshot_1](https://user-images.githubusercontent.com/54905232/88428211-1de22000-cdfd-11ea-890e-19cf5fdb815f.png)

The process done here: 
sends the key value as parameter to the decryption method and gets the desired string.

### Things To Do :
- Installing Python libraries (and including dnlib.dll)
- Defining string decryption method as name and MethodInfo
- Determining string encrypted places and getting decrypted value by using Invoke
- Placing the received decrypted value
- Saving

### Import and Load

First of all, i will need install pythonnet library

> pip install pythonnet

Let's import the reflection libraries.

{% highlight python %}
import clr
from System.Reflection import Assembly, MethodInfo, BindingFlags
from System import Type
{% endhighlight %}

In the main block, we will add our dnlib.dll file as a reference, then we will load the obfuscated application to our variables named module and assembly.

{% highlight python %}
def DecryptStrings():
    MethodIdentifier()


def Save():
    module.Write("deobfuscated.exe")


def MethodIdentifier():
    pass


if __name__ == "__main__":
    clr.AddReference(r"DNLİB PATH")
    import dnlib
    from dnlib.DotNet.Emit import OpCodes
    global module # her fonksiyondan erişebilir olması için
    global assembly
    module = dnlib.DotNet.ModuleDefMD.Load("WindowsFormsApp3.exe")
    assembly = Assembly.LoadFrom("WindowsFormsApp3.exe")
    MethodIdentifier()
    DecryptStrings()
    Save()
{% endhighlight %}

> Let me give a small explanation of why we are loading two modules separately? assembly, was loaded from Reflection library for Invoke operation (this feature is not available in dnlib). module, for editing operations.

## Decryption Method

In this section, our script will only find decryption method then save to stringMethodName and invokeMethod variables.

Let's reopen and analyze it with dnSpy, let's see how the decryption method has distinctive features. 

The decryption method is here:
![Screenshot_3](https://user-images.githubusercontent.com/54905232/88432077-33a71380-ce04-11ea-9788-3ca06917535d.png)

Now let's move on to the IL section, which the script can read. (Right Click > Edit IL Instructions)
![Screenshot_4](https://user-images.githubusercontent.com/54905232/88432269-897bbb80-ce04-11ea-8f12-252681cbbf37.png)

The most distinctive features of the method:
- Lenght of IL Instructions
- 0th OpCode and 2th OpCode

> IL Instructions lenght = 107
> 0th OpCode = LdcI4
> 2th OpCode = Ldsfld

{% highlight python %}
def MethodIdentifier():
    global stringMethodName #1
    global invokeMethod #1
    eFlags = BindingFlags.Static | BindingFlags.Public | BindingFlags.NonPublic #2
    for type in module.Types: #3
        for method in type.Methods:
            if not method.HasBody: pass #4
            if len(method.Body.Instructions) < 107: pass #5
            i = 0
            while i < len(method.Body.Instructions): #6
                if(method.Body.Instructions[i].IsLdcI4() and
                     method.Body.Instructions[i + 2].OpCode == OpCodes.Ldsfld): #7
                    stringMethodName = str(method.Name)
                    stringTypeName = str(type.Name)
                    for types in assembly.GetTypes(): #8 ...
                        if types.Name == stringTypeName:
                            classInstance = types
                            break

                    for methods in classInstance.GetMethods(eFlags):
                        if methods.Name == stringMethodName:
                            invokeMethod = methods
                            return
                    break
                i += 1
{% endhighlight %}

Kodumuzu açıklayalım :

Code description

- 1) We define our two variables with global keywords so that other functions can access our variables.
- 2) https://docs.microsoft.com/en-US/dotnet/api/system.reflection.bindingflags?view=netcore-3.1
- 3) We navigate between the Types of the module we installed (ie obfuscated binary), that is, Classes. Just below, in the methods of types…
- 4) We call method.BODY.INSTRUCTIONS in the rest of the code. Therefore, if we do not have a body, we provide the skip operation to avoid errors.
- 5) Yes, here we have our first condition, if your IL instructions lenght is less than 107, pass that method.
- 6) We move between ILs to ensure our second condition
- 7) If your currently instruction is LdcI4 (LdcI4.1, LdcI4.2 [...], we use the IsLdcI4() function to simplify it.) and after opCode is Ldsfld then...
- 8) We assign the classInstance assignment to find the decryption method later. In order to get the method to be invoked, ie the Decryption method, in the "MethodInfo" type, we are doing the procedures here.

## Decryptor Method

After finding our decryption method, all we do is find the encrypted fields and send the invoking parameter to the decryption method and invoke it. We place the resulting result in its place.

The first thing we need to do is get to know the obfuscated regions. We open and look at dnSpy.
![Screenshot_5](https://user-images.githubusercontent.com/54905232/88434258-7ff45280-ce08-11ea-95a1-ef0682f21df7.png)

When we look at the IL instructions of an obfuscated place, we will only see that a numeric expression is sent to the method.
But multiple methods can work like this. Here, the “stringMethodName” variable that we defined earlier in this section will be useful for us.
In the Operand of the section with "call" OpCode, we can say that this is an obfuscated field if "stringMethodName" is contained.

The most distinctive features of this :
- If your current index is LdcI4 and the next instruction OpCode is call
- If the Operand of the instruction with call OpCode contains "stringMethodName"

Code :

{% highlight python %}
    decryptedStrings = 0 # Ne kadar stringi decrypt ettiğini görelim burda
    for type in module.Types: # Klasik dolaşma işlemleri
        if not type.HasMethods: pass # Eğer metodu yoksa atla
        
        for method in type.Methods:
            if not method.HasBody: pass
            i = 0
            operText = ""
            while i < len(method.Body.Instructions):
                operText = str(method.Body.Instructions[i].Operand).encode()
                if(method.Body.Instructions[i].OpCode == OpCodes.Call and
                        operText.find(str(stringMethodName).encode()) != -1):
.........
{% endhighlight %}

What we now need to do is pull the value of what we are going to send as an argument and invoke and replace the incoming value:

{% highlight python %}
  keyValue = method.Body.Instructions[i - 1].GetLdcI4Value()
  result = str(invokeMethod.Invoke(None, [keyValue]))
  method.Body.Instructions[i - 1].OpCode = OpCodes.Nop
  method.Body.Instructions[i].OpCode = OpCodes.Ldstr
  method.Body.Instructions[i].Operand = result
  decryptedStrings += 1
i += 1
{% endhighlight %}

Required files and full code:
> https://github.com/Rhotav/Python-Method-Invoker
