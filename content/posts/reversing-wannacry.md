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
00408140    int32_t main()

00408155        void variableContainingURL
00408155        char* AnotherVariableContainingURL
00408155        char* VariableContainingURL
00408155        VariableContainingURL, AnotherVariableContainingURL =
00408155            __builtin_memcpy(dest: &variableContainingURL, 
00408155            src: "
00408155                http://www.iuqerfsodp9ifjaposdfjhgosurijfaewrwergwea.com", 
00408155            count: 56)
00408157        *VariableContainingURL = *AnotherVariableContainingURL
00408158        int32_t var_17
00408158        __builtin_memset(dest: &var_17, ch: 0, count: 23)
0040817b        int32_t hInternet = InternetOpenA(lpszAgent: nullptr, 
0040817b            dwAccessType: 1, lpszProxy: nullptr, lpszProxyBypass: nullptr, 
0040817b            dwFlags: 0)
00408194        int32_t hInternet_1 = InternetOpenUrlA(hInternet, 
00408194            lpszUrl: &variableContainingURL, lpszHeaders: nullptr, 
00408194            dwHeadersLength: 0, dwFlags: 0x84000000, dwContext: 0)
00408194        
004081a5        if (hInternet_1 != 0)
004081bc            InternetCloseHandle(hInternet)
004081bf            InternetCloseHandle(hInternet: hInternet_1)
004081c8            return 0
004081c8        
004081a7        InternetCloseHandle(hInternet)
004081ab        InternetCloseHandle(hInternet: 0)
004081ad        stage2()
004081b9        return 0
```



