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

        VariableContainingURL, AnotherVariableContainingURL =
            __builtin_memcpy(dest: &variableContainingURL, 
            src: "
                http://www.iuqerfsodp9ifjaposdfjhgosurijfaewrwergwea.com", 
            count: 56)

        *VariableContainingURL = *AnotherVariableContainingURL
        int32_t var_17

        __builtin_memset(dest: &var_17, ch: 0, count: 23)
        int32_t hInternet = InternetOpenA(lpszAgent: nullptr, 
            dwAccessType: 1, lpszProxy: nullptr, lpszProxyBypass: nullptr, 
            dwFlags: 0)

        int32_t hInternet_1 = InternetOpenUrlA(hInternet, 
            lpszUrl: &variableContainingURL, lpszHeaders: nullptr, 
            dwHeadersLength: 0, dwFlags: 0x84000000, dwContext: 0)
        
        if (hInternet_1 != 0)
            InternetCloseHandle(hInternet)
            InternetCloseHandle(hInternet: hInternet_1)
            return 0
        
        InternetCloseHandle(hInternet)
        InternetCloseHandle(hInternet: 0)

        stage2()

        return 0
```



