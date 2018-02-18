# IniFiles


## Overview

INI files are still useful to provide settings to an application. Neither Vista nor Windows 7 are going to change this.

The Windows API methods provided to read a particular value have one advantage: they deliver always up-to-date values. Whether that is an important feature or not is another matter.

They have disadvantages as well:

 * They are slow
 * They return everything as a string

In case nobody else is changing your INI files while your application is running then the `IniFile` class introduced in this article might attract your attention.

This class allows you to use a kind of APL-Syntax in your INI files. Values not enclosed in quotes will be converted to numbers, everything else gets a string.


## The INI specification

INI files are not well-defined. Different implementation come with all sorts of specialties. For details see:
http://en.wikipedia.org/wiki/INI_files

However, there is some common ground:

* In old fashioned INI files values don't have to be enclosed in quotes. The Windows API functions however except quotes; if found they are simply removed.

* It is widely agreed that neither section names nor value names should carry a blank, although the Windows API functions accept them. The `IniFiles` class would throw an error if it finds one in any name.


## Differences between APL-like INI files and classic ones

* While values within quotes are treated as text those without quotes are treated as numbers. 

* Numeric values can carry either a single value or a vector of values.

* Nested strings are possible.

* There is the concept of variables which one can refer to in any part of an INI file once they got defined.

* Section names must be valid APL names.

* Value names must be valid APL names. That implies that they may not carry a blank.


## A warning

An INI file is by definition not a kind of database and should ''not'' be used as such. Therefore it is recommended to use the `Save` method only for two purposes:

* Establish a new INI file with default settings.

* Convert an old-fashioned INI file into an APL-like INI file.


## Details

### Character values

An entry like:

```
      HomeFolder='C://Appl/'
```

results in a string holding the path.


### Numeric values

An entry like:

```
      FormSize=300 400
```

results in a two-element-vector "!FormSize" holding two integers.


### References (place holders)

If you need to define a number of paths like:

```
path='C:\MyApp'
pathCertificates='C:\MyApp\Certificates'
pathRootCertificates='C:\MyApp\RootCert'
pathLogfiles='C:\MyApp\Log'
```

you can simplify this by using the "replacement" syntax:

```
path='C:\MyApp'
pathCertificates='{path}\Certificates'
pathRootCertificates='{path}\RootCert'
pathLogfiles='{path}\Log'
```

Naturally "path" must be specified upfront. Prior to version 1.5, this must be specified within the same section. As a result the same variable needed to be specified more than once if the same path needed to be available in more than one section.

Since version 1.5 this restriction was lifted by the introduction of "local" variables, see there.

Note that you can still specify `{}` as part of the data: just double them like quotes:

```
      '{{⍵/⍨2=+⌿0=⍵∘.|⍵}⍳⍵}' ←→ '```{⍵/⍨2=+⌿0=⍵∘.|⍵}}⍳⍵}}'
```

### Local variables 

Local values are those specified above the first section. They have only one purpose: to be used as references in several sections.

There are some restrictions:

 * They can only be used during the instantiation
 * They must not be nested
 * Although it is possible to specify a numeric value this does not make any sense since numeric values cannot be used as references

### Nested entries: special "Add" syntax 

Sometimes one needs to specify potentially long lists on a particular keyword, for example a list of IP addresses a server is supposed to ignore: "DenyIP". Specifying them on one single line is very hard to read and prone to error in case of a change.

Instead this can be achieved with a special syntax:

```
DenyIP=''
DenyIP,='2001:0db8:0000:0000:0000:0000:1428:57ab'
DenyIP,='2001:0db8:0000:0000:0000::1428:57ab'
```

This results in a nested vector of strings. Note that you '''must''' initialize the vars as an empty vector in the first place. The `=,` syntax implies an `⊂` (enlose) on the data to the right of "=".

This syntax is supported for both, characters and numbers:

```
vector=''
vector,=1 2 3
vector,=200 300
```

leads to:

```
(1 2 3) (200 300)
```

### Importing another INI file 

Above the first section definition one can also import another INI file. This can be used for two different purposes:
 * Make an INI file path/drive independent, for example in order to support an application on a notebook as well as a server.
 * Import a network based INI file from a local one.

This is how an INI file that is going to be imported elsewhere could look like:

```
; INI file to be imported
[DRIVES]  ; This is a comment
ArchiveDrive='D:\'
DocDrive='M:\'
```

Assuming that the name of this INI file is "local.ini":

```
!Import local.ini
[PATHS]
ArchivePath='{ArchiveDrive}this\that\'
DocPath='{DocDrive}there\'
```

Using this technique all values that depend on the current environment can be specified in `local.ini` while all the other entries can be specified in the second INI file. 

However, with version 2.0.0 the constructor accepts more than one filename: use this to effectively merge INI files. 

Which technique is more appropriate depends on the application. See [Merging versus importing](#) for a discussion.

### Classic INI files 

Since version 2.2 `IniFiles` is able to process classic INI files. This means that these two values:

```
text=hello
digits=1 2 3
```

are both converted into text. 

Note that an INI file must be consistent: it cannot mix classic stuff with new stuff. This INI file:

```
text1=hello
test2='universe'
```

would therefore cause an error. This means that in order to be identified as classic an INI file must not enclose any of its values in quotes.

This INI file would not work either:

```
text1=
test2='universe'
```

because it also is inconsistent but this one would:

```
text1=
test2=universe
```

"text1" ends up as an empty text vector.

Note that an INI file that contains any local variables or that contains at least one line that starts with "!Import" before the first section is not considered to be a classic INI file.

#### Ambiguity 

Note that there is room for ambiguity; consider this INI file:

```
[CONFIG]
no1=1
no2=2
no3=3
```

Interpreted as a classic INI file this would result in three variables carrying the strings "1", "2" and "3". Interpreted as an APL-style INI file you would get three variables carrying integer values.

In case of ambiguity the INI file is interpreted as an APL-style one.

#### The "OldStyleFlag" 

From version 2.2 on there is a new property `OldStyleFlag` which is a Boolean that will be 1 only if the processed INI file is a classic one.

#### Converting Classic INI files 

An INI file can be converted into an APL-like INI file by instantiating it and then issuing the `Save` command. Note that all values will be surrounded by quotes by then even if a value "Number" carries "123" as value. If you want to take advantage of the numeric data type you have to make changes to the INI file. Of course this means that you also have to change your code.

### Check whether an INI file has changed 

When instantiated a read-only property `EstablishedAt` is set. This is used by the method `HasInifileChanged` which returns a 1 in case the INI file had been changed since it was instantiated, otherwise 0.

## Examples 
### Creating an Instance 
After creating an instance from the class:

```myIni←⎕New #.IniClass (,⊂'C:/Appl/Example.ini')```

....

### Accessing Data with the "Get" method 

You can get all information you are interested in by calling the method "Get". Note that names are '''not''' case sensitive.

(Note that any instance of the `IniFiles` class can be converted to an ordinary namespace - see the `Convert` method)

Given this file "Example.ini":

```
[GENERAL]
MaxNoOfErrors=20
FormSize=800 1200
LogfileFlag=1
LogLevels=1 2 3 ; from 1 to 9

[DIR]
Home='C:/mainfolder/'
AppFolder='{Home}appls/'
DocsFolder='{Home}docs/'
LogFileFolder='{Home}Logs/'
```
You can get any level of information you are interested in:

 * get everything
 * get all keys and values of a particular section
 * get a particular value from a particular section

#### Examples with "Get" 
```
      myIni.Get ⍬ ⍬
 GENERAL
          MAXNOOFERRORS                    20
          FORMSIZE                   800 1200
          LOGFILEFLAG                       1
          LOGLEVELS                     1 2 3
 DIR
          HOME                 C:/mainfolder/
          APPFOLDER      C:/mainfolder/appls/
          DOCSFOLDER      C:/mainfolder/docs/
          LOGFILEFOLDER   C:/mainfolder/Logs/
      myIni.Get'General' ⍬
MAXNOOFERRORS        20
FORMSIZE       800 1200
LOGFILEFLAG           1
LOGLEVELS         1 2 3
      myIni.Get'General' 'FormSize'
800 1200
      ¯1 myIni.Get'General' 'Unknown' ⍝ with default
¯1
      myIni.Get'General' 'Unknown' ⍝ without default
Value Error: "Unknown"
myDoc.Get'General' 'Unknown'
```
### Indexing 
Since version 1.1, the class provides a default property. That means you can access values by indexing.

Examples (with the same INI file listed above):

```
      myIni[⊂'GeneRAL:']
20  800 1200  1  1 2 3
            ⊃myIni[⊂'GeneRAL:FormSize']
800 1200
```
### Assigning 
 .`myIni[⊂'GeneRAL:FormSize']←⊂12 23`

### The "Put" method 
 .` (12 23) myIni.Put 'GeneRAL:FormSize'`

### The "Save" method 
You can also change a particular value but the changed value will persist only if you execute the "Save" method at some point:

```
      myIni[⊂'GeneRAL:FormSize']←⊂'¯1 1000
      myIni.Save
```

You can use the `Save` method to convert an old-fashioned INI file because every value will be saved within quotes. There is an important restriction however: The `Save` method cannot be invoked when more than one file was evaluated. See [Multiple INI files ](#) for details.

### The "GetSections" method 

With version 2.8.0 a method `GetSections` was introduced. It returns all sections names in uppercase letters:

```
      myIni.GetSections
GENERAL
DIR
```

### The "Convert" method 

Any instance of the `IniFiles` class can be converted into an ordinary namespace. The `myIni` instance created earlier on for example can be converted in two ways:

#### Use namespace rather than instance 

The `Convert` method can be used to convert an instance into an ordinary namespace. It takes a reference to a namespace as the right argument:

```
      Ini←myIni.Convert ⎕NS''
      Ini.List ⍬
 DIR      AppFolder      C:/mainfolder/appls/ 
 DIR      DocsFolder      C:/mainfolder/docs/ 
 DIR      Home                 C:/mainfolder/ 
 DIR      LogFileFolder   C:/mainfolder/Logs/ 
 GENERAL  FormSize                   800 1200 
 GENERAL  LogLevels                     1 2 3 
 GENERAL  LogfileFlag                       1 
 GENERAL  MaxNoOfErrors                    20 
      Ini.GENERAL.MaxNoOfErrors
20
```

You can ask `Convert` to convert the instance into a flat namespace, effectively ignoring the sections:

```
      Ini←'flat' myIni.Convert ⎕NS''
      Ini.List ⍬
      Ini.List ⍬
 AppFolder      C:/mainfolder/appls/    
 DocsFolder      C:/mainfolder/docs/    
 FormSize                   800 1200    
 Home                 C:/mainfolder/    
 LogFileFolder   C:/mainfolder/Logs/    
 LogLevels                     1 2 3    
 LogfileFlag                       1    
 MaxNoOfErrors                    20 
      Ini.MaxNoOfErrors
20
```


### Multiple INI files (Merging)

Note that since version 2.0.0 one can specify more than one filename in the `⎕NEW` statement. This effectively merges the INI files together. Note that....

 1. in case of name clashes the last file wins.
 1. you cannot execute the `Save` method when more than one INI file was specified.
 1. the `IniFilename` property always returns a simple string. In case of more than one INI file the filenames are separated by ";".

Note that placeholders are replaced only after all INI files specified have been processed. Imagine this general INI file `Foo.INI`:

```
[CONFIG]
Home='C:\'

[PATHS]
PrintFolder='{Home}ThePrintFolder'
```

and a machine-specific INI file `Foo_MyMachine.INI`:

```
[CONFIG]
Home='D:\SomeWhereElse'
```

When only the first INI file is processed and the result of `⎕NEW` is assigned to `MyIni` then you get this:

```
      MyIni.PrintFolder
C:\ThePrintFolder
```

When you process both INI file, first `Foo.INI` and then `Foo_MyMachine.INI` then you get this:

```
      MyIni.PrintFolder
D:\SomeWhereElse
```

That way you can specify important defaults at a client's side in the main INI file. When running the same application on your laptop you can add a file that carries the necessary differences.

There is a shared method `GetIniFiles` available that supports this:

#### The GetIniFiles method 

If a directory "foo\" contains just a file "my.ini" then:

```
      #.IniFiles.GetIniFiles 'foo\my'
foo\my.ini
      #.IniFiles.GetIniFiles 'foo\my.ini'
foo\my.ini
```

If there are four files: "my.ini", "my2.ini", "my_kai.ini" and "my_peter.ini" then on a computer with the name "kai":

```
   #.IniFiles.GetIniFiles 'foo\my'
foo\my.ini
foo\my_kai.ini
```

but on a computer with the name "peter":

```
      #.IniFiles.GetIniFiles 'foo\my'
foo\my.ini
foo\my_peter.ini
```



### Merging versus importing 

Merging two or more INI files shares many features with importing them. Let's discuss the differences.

#### Local specialities 

Imagine that at your clients site you run a certain application from a certain folder on a certain server. You specify a "Home" value in your master INI file called "Master.INI" to reflect this:

```
[CONFIG]
Home='\\MyServer\MyApp'
```

You might want to run and develop that application on your laptop as well. On your laptop it's located in a folder "Foo" on your local D:\ drive. Let's assume that you laptop has the name "myLaptop".

By creating a second INI file called "Master_myLaptop.INI" with this contents:

```
[CONFIG]
Home='D:\Foo\'
```

you can solve the problem. Since version 2.6 there is a method `GetIniFiles` available to deal with this - see above.

#### Multiple locations 

I found myself once in a situation where I had to keep the same INI file on different computers on the network. Of course this is dangerous because to keep them in sync is important but will at one stage or another almost certainly fail. `!Import` to the rescue: you keep just one master file while all the other INI files contain just one line pointing to that master file:

```
!Import  \\servername\INIs\master.ini
```

## Editing INI files 

--(Note that there is a tailored INI-file editor available as part of the APLTree project. See !EditIni for details.)--

The !EditIni project has been retired.