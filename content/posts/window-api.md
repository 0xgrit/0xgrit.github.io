# Da Windows API
The Windows Application Programming Interface is the user-mode system programming inrterface tfor the Windows Os family. Back then its called Win32 API to exlude it from versions, but we're just going to call it WinAPI.

## WinAPI Flavors
WinAPI consted of C-style functions only becuaes its the porgramming langauge that is the closest to expose OS srevices, but it had ownsides like lack of naming consistency, and logicalgroupings. resulting in difficulties resulted in some newer APIs using different API mechanism like the Component Object Model.

COM was made to enable MS aps to communicate and exhcnage data berween documents (e.g putting word document inside pwoer point) thisi s called Object Linking and Embedding that is made using an old IWindows messaging mechanism called Dynamic Data eExchnage, was limited but another one is developed which is OLE 2.

COM is based on 2 foudnational principles:
### First principle: Interface-based communication
Clients interact with COM objects through interfaces. An interface is essentially a contract, a defined set of related methods that the object promises to provide. These interfaces are organized using the virtual table (vtable) mechanism, which is the same technique C++ compilers use to handle virtual functions. This approach solves two major problems: it achieves binary compatibility across different compilers, and it eliminates name mangling issues. As a result, COM objects can be called from virtually any programming language—C, C++, Visual Basic, .NET languages, Delphi, and many others—regardless of which compiler was used.


### Second principle: Dynamic loading
COM components are loaded at runtime rather than being statically linked into the client application. The actual implementation lives in what's called a COM server—typically a Dynamic Link Library (DLL) or executable (EXE) file. When your program needs the component, it loads it dynamically.

Beyond these fundamentals, COM includes features for security, cross-process communication (marshalling), threading models, and more. 
