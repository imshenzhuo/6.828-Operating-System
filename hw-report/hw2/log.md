# 重定向

## 输入重定向

``` cpp
cat < file
```

```c
char *argv[2];
argv[0] = "cat";
argv[1] = 0;
if(fork() == 0) {
	close(0);
	open("input.txt", O_RDONLY);
	exec("cat", argv);
}
```

总是从`fd=0`读

### 输出重定向

``` bash
echo "echoooo" > x.txt
```

``` cpp
if(fork() == 0) {
	close(1);
	open("x.txt", O_WTONLY);
	exec("echo", argv);
}
```

