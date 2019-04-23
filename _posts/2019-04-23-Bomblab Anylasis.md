# Bomblab Analysis

#### 1. 主函数及辅助调用函数

##### a. <Main>

```python
  400ee4:	48 89 c7             	mov    %rax,%rdi
  400ee7:	e8 a4 00 00 00       	callq  400f90 <phase_1>
  400eec:	e8 4e 09 00 00       	callq  40183f <phase_defused>
  400ef1:	bf 38 26 40 00       	mov    $0x402638,%edi
  400ef6:	e8 c5 fc ff ff       	callq  400bc0 <puts@plt>
  400efb:	e8 19 08 00 00       	callq  401719 <read_line>
```

在主函数的主体部分，每个阶段的炸弹都对应着如上所示的类似代码。先调用`<phase_?>`函数，是拆炸弹必需的函数，然后调用`<phase_defused>`函数，若输入的密钥正确则使用`<puts@plt>`打印相关的语句，不正确的话会跳转到炸弹所在的地址引起爆炸。最后利用`<read_line>`函数继续读取下一个密钥。需要注意的是，`<puts@plt>`是一个过程链接库，类似于动态链接库，会调用栈中的其他函数，这些函数在反汇编代码中并没有具体给出实现代码。涉及到的具体函数有`<puts@plt>`（输出打印）, `<_isoc99_sscanf@plt>`（读入）, `<strtol@plt>`（将字符串转换成长整型），这些函数都用在后面各个phase中。还有一些辅助函数，`<strings_not_equal>`, `<string_length>`, `<read_six_numbers>`, `<explode_bomb>`，用于多个phase中，接下来依次介绍。

##### b. <Phase_defused>

此函数在每次调用一个phase后都会调用，其中有一段代码是：

```python
  40185d:	83 3d 58 2f 20 00 06 	cmpl   $0x6,0x202f58(%rip) # 6047bc <num_input_strings>
  401864:	75 6d                	jne    4018d3 <phase_defused+0x94>
```

这是在比较目前为止输入的字符串的总数目，与6比较，不相等则直接跳到最后一部分。其含义是：因为我们要破解6个炸弹，每破解好一个都会看看是否已经破解完6个，没有破解完则继续破解，破解完了就要做点其他事情了。有后面的代码可知，这里的“其他事情”是指：是否有“资格”进入到secret_phase继续破解秘密炸弹。具体的如何获取“资格”在secret_phase部分介绍。

##### c. <string_length>

```python
00000000004013ab <string_length>:
  4013ab:	80 3f 00             	cmpb   $0x0,(%rdi) 					    # rdi是字符串的起始地址
  4013ae:	74 12                	je     4013c2 <string_length+0x17>      # 这是在查看字符串是否为空，为空则返回eax=0，即字符串长度为0
    
  4013b0:	48 89 fa             	mov    %rdi,%rdx						# rdx是遍历字符串的指针，初始化指向第一个字符
    
  4013b3:	48 83 c2 01          	add    $0x1,%rdx						# 指针每次加一
  4013b7:	89 d0                	mov    %edx,%eax						# 令eax指向当前地址，与edi首地址求差值，结果即为：当前正在处理的是第几个字符
  4013b9:	29 f8                	sub    %edi,%eax						# 
  4013bb:	80 3a 00             	cmpb   $0x0,(%rdx)						# 若edx的内容为0，即字符串遍历完毕，说明刚才存储到eax的差值即为字符串的长度，返回。
  4013be:	75 f3                	jne    4013b3 <string_length+0x8>		# 若不为0，则将指针edx后移一位循环上述步骤。
  4013c0:	f3 c3                	repz retq 
  4013c2:	b8 00 00 00 00       	mov    $0x0,%eax
  4013c7:	c3                   	retq 
```

此函数的作用：求 以`rdi`作为起始地址的 字符串的长度，最后的结果（长度）保存在`eax`中返回。

##### d. <strings_not_equal>

**第一段**

```python
00000000004013c8 <strings_not_equal>:
  4013c8:	41 54                	push   %r12
  4013ca:	55                   	push   %rbp
  4013cb:	53                   	push   %rbx						# 新建堆栈，堆栈初始化            				
  4013cc:	48 89 fb             	mov    %rdi,%rbx				
    
  4013cf:	48 89 f5             	mov    %rsi,%rbp				# 先将一个字符串a的起始地址esi保存起来，放在rbp中
  4013d2:	e8 d4 ff ff ff       	callq  4013ab <string_length>	# 调用此函数 求edi为起始地址的字符串（另一个字符串b）的长度，存入eax中
  4013d7:	41 89 c4             	mov    %eax,%r12d				# 将b的长度存入r12中
    
  4013da:	48 89 ef             	mov    %rbp,%rdi				# 将字符串a的起始地址存入edi中
  4013dd:	e8 c9 ff ff ff       	callq  4013ab <string_length>	# 求字符串a的长度，放在eax中
  4013e2:	ba 01 00 00 00       	mov    $0x1,%edx				# 函数返回前先将edx置1
  4013e7:	41 39 c4             	cmp    %eax,%r12d				# 将eax和r12的值比较，即比较a和b字符串的长度，不相等则跳到40142b。
  4013ea:	75 3f                	jne    40142b <strings_not_equal+0x63>
```

本段代码的作用：比较以`edi`为起始地址的字符串和`esi`为起始地址的字符串的长度是否相等。

```python
  4013ec:	0f b6 03             	movzbl (%rbx),%eax						# 若两个字符串的长度相等，初始时rbx存储的是字符串b的起始地址，就会将第一个字符存入eax中
  4013ef:	84 c0                	test   %al,%al							# 测试al中的字符是否为空，若为空则说明两个字符串都为空
  4013f1:	74 25                	je     401418 <strings_not_equal+0x50>	# 由401418的代码，返回eax=0，说明两个字符串相等
    
    																		# rbp一开始已经存入了esi，字符串a的起始地址。
  4013f3:	3a 45 00             	cmp    0x0(%rbp),%al					# 将al中的字符与rbp的地址对应的字符比较
  4013f6:	74 0a                	je     401402 <strings_not_equal+0x3a>  # 若相等则跳到401402继续比较下一个数
  4013f8:	eb 25                	jmp    40141f <strings_not_equal+0x57>
    
  4013fa:	3a 45 00             	cmp    0x0(%rbp),%al					# 比较al中的字符与edp对应的字符，即比较ebx与ebp的地址对应的字符
  4013fd:	0f 1f 00             	nopl   (%rax)							# 若不相等，由下面的代码可知，返回eax=1
  401400:	75 24                	jne    401426 <strings_not_equal+0x5e>
    
  401402:	48 83 c3 01          	add    $0x1,%rbx						# 比较下面的字符：
  401406:	48 83 c5 01          	add    $0x1,%rbp						# 将两个字符串的指针rbx与rbp后移一位
  40140a:	0f b6 03             	movzbl (%rbx),%eax						# rbx地址对应着的字符移入eax中
  40140d:	84 c0                	test   %al,%al							# 若eax不是空字符，说明比较没结束
  40140f:	75 e9                	jne    4013fa <strings_not_equal+0x32>  # 则继续比较下一个对应的字符
    
  401411:	ba 00 00 00 00       	mov    $0x0,%edx						# 若eax是空字符
  401416:	eb 13                	jmp    40142b <strings_not_equal+0x63>	# 说明字符串比较完毕，由下面的代码可知，返回eax=0
    
  401418:	ba 00 00 00 00       	mov    $0x0,%edx						# edx=0，跳到40142b使eax=0，返回
  40141d:	eb 0c                	jmp    40142b <strings_not_equal+0x63>	#
    
  40141f:	ba 01 00 00 00       	mov    $0x1,%edx						# 使edx=1，返回eax=1
  401424:	eb 05                	jmp    40142b <strings_not_equal+0x63>
    
  401426:	ba 01 00 00 00       	mov    $0x1,%edx
    
  40142b:	89 d0                	mov    %edx,%eax						# 将edx的值存入eax中，返回
  40142d:	5b                   	pop    %rbx
  40142e:	5d                   	pop    %rbp
  40142f:	41 5c                	pop    %r12
  401431:	c3                   	retq  
```

由上面的分析可知，这个函数会比较`esi`和`edi`分别为起始地址对应的字符串是否相等。若相等则返回eax=0，若不相等则返回eax=1。

##### e. <read_six_numbers>

```python
00000000004016d7 <read_six_numbers>:
  4016d7:	48 83 ec 18          	sub    $0x18,%rsp
  4016db:	48 89 f2             	mov    %rsi,%rdx						# 将读入的第一个数据的地址备份到rdx中
  4016de:	48 8d 4e 04          	lea    0x4(%rsi),%rcx					# rcx存储第二个数的地址
  4016e2:	48 8d 46 14          	lea    0x14(%rsi),%rax					# rax存储第6个数的地址
  4016e6:	48 89 44 24 08       	mov    %rax,0x8(%rsp)					# 将第6个数，即最后一个数的地址存入（rsp+8）单元中
  4016eb:	48 8d 46 10          	lea    0x10(%rsi),%rax					# rax存储第5个数的地址
  4016ef:	48 89 04 24          	mov    %rax,(%rsp)						# 将第5个数的地址存入（rsp）单元中，注意此时的rsp和此函数返回到原程序的
  4016f3:	4c 8d 4e 0c          	lea    0xc(%rsi),%r9					# r9存储第4个数的地址 （rsi+12）
  4016f7:	4c 8d 46 08          	lea    0x8(%rsi),%r8					# r8存储第3个数的地址 （rsi+8）
    
  4016fb:	be a1 29 40 00       	mov    $0x4029a1,%esi					# 将esi和eax初始化，准备读取数据
  401700:	b8 00 00 00 00       	mov    $0x0,%eax						# 
  401705:	e8 a6 f5 ff ff       	callq  400cb0 <__isoc99_sscanf@plt>		# 调用scanf函数读取数据
  40170a:	83 f8 05             	cmp    $0x5,%eax						# 若读入的数字的数目大于5
  40170d:	7f 05                	jg     401714 <read_six_numbers+0x3d>	# 说明读入的数据超过了6个
  40170f:	e8 8d ff ff ff       	callq  4016a1 <explode_bomb>			# 引爆炸弹
  401714:	48 83 c4 18          	add    $0x18,%rsp						# 
  401718:	c3                   	retq   
```

调用此函数会读取6个数字，并且必须保证不能超过6个数字，否则会爆炸。读取结束后，第一个数的地址存放在`rdx`中。

##### f. <explode_bomb>

```python
00000000004016a1 <explode_bomb>:
  4016a1:	48 83 ec 08          	sub    $0x8,%rsp
    
  4016a5:	bf 81 29 40 00       	mov    $0x402981,%edi
  4016aa:	e8 11 f5 ff ff       	callq  400bc0 <puts@plt>
  4016af:	bf 8a 29 40 00       	mov    $0x40298a,%edi
  4016b4:	e8 07 f5 ff ff       	callq  400bc0 <puts@plt>
    
  4016b9:	bf 00 00 00 00       	mov    $0x0,%edi
  4016be:	e8 d2 fe ff ff       	callq  401595 <send_msg>
    
  4016c3:	bf 30 28 40 00       	mov    $0x402830,%edi
  4016c8:	e8 f3 f4 ff ff       	callq  400bc0 <puts@plt>
    
  4016cd:	bf 08 00 00 00       	mov    $0x8,%edi
  4016d2:	e8 19 f6 ff ff       	callq  400cf0 <exit@plt>
```

若调用了此函数，根据`<puts@plt>`函数依次输出0x402981，0x40298a中的字符串；再利用`send_msg`函数发送爆炸信息；再输出0x402930中的内容，最后调用`<exit@plt>`函数退出程序。

#### 2. <Phase_1>

```python
0000000000400f90 <phase_1>:
  400f90:	48 83 ec 08          	sub    $0x8,%rsp
  400f94:	be 90 26 40 00       	mov    $0x402690,%esi
  400f99:	e8 2a 04 00 00       	callq  4013c8 <strings_not_equal>
  400f9e:	85 c0                	test   %eax,%eax
  400fa0:	74 05                	je     400fa7 <phase_1+0x17>
  400fa2:	e8 fa 06 00 00       	callq  4016a1 <explode_bomb>
  400fa7:	48 83 c4 08          	add    $0x8,%rsp
  400fab:	c3                   	retq  
```

第三行调用比较字符串的函数`strings_not_equal`，会更改eax的值，若`test %eax, %eax`的值为0，说明字符串相等，SF=0，（je满足）此时会跳过炸弹，所以要求输入的字符串与内存中0x402690的字符串相等。根据命令 `x/s 0x402690` 得到字符串 `“There are rumors on the internets.”`，所以应输入`There are rumors on the internets.`

#### 3. <Phase_2>

**第一段**

```python
0000000000400fac <phase_2>:
  400fac:	55                   	push   %rbp
  400fad:	53                   	push   %rbx
  400fae:	48 83 ec 28          	sub    $0x28,%rsp                   # 此处是堆栈的准备，留出0x28个“位置”，rsp和rsi同时指向堆栈的
  400fb2:	48 89 e6             	mov    %rsp,%rsi                    # 底部，即第一个要输入的信息的位置。
  400fb5:	e8 1d 07 00 00       	callq  4016d7 <read_six_numbers>    # 调用此函数读取6个数字，从下往上放入刚空出来的堆栈里面
  400fba:	83 3c 24 00          	cmpl   $0x0,(%rsp)                  # 比较第一个输入的数字与0，只有当此数字为0，SF=0，才满足jns，
  400fbe:	79 24                	jns    400fe4 <phase_2+0x38>        # 跳过炸弹。故输入的第一个数为0.
  400fc0:	e8 dc 06 00 00       	callq  4016a1 <explode_bomb>
```

如注释中，此段代码要求输入6个数字，而且第一个数字应为0。

**第二段**

```python
  400fc5:	eb 1d                	jmp    400fe4 <phase_2+0x38>
        
  400fc7:	89 d8                	mov    %ebx,%eax 			  # ebx的值赋给eax，第一次循环eax=1
  400fc9:	03 45 fc             	add    -0x4(%rbp),%eax		  # eax中的数字加上rbp指向的前一个数字的值。比如第一次循环，会加上第一个数
  400fcc:	39 45 00             	cmp    %eax,0x0(%rbp)		  # 比较rbp对应的数字和eax，第一次的话（eax=0+1=1）
  400fcf:	74 05                	je     400fd6 <phase_2+0x2a>  # 必须保证二者相等。
  400fd1:	e8 cb 06 00 00       	callq  4016a1 <explode_bomb>  # 不然会爆炸的。所以rbp对应的数字即第二个数要等于1。
  400fd6:	83 c3 01             	add    $0x1,%ebx			  # ebx在每次循环中加1，依次变为2，3，4...
    															  # 这样的话每次循环，eax就会不断加上2，3，4...
  400fd9:	48 83 c5 04          	add    $0x4,%rbp			  # rbp指向下一个数字，第一次循环（指向第三个数字）
  400fdd:	83 fb 06             	cmp    $0x6,%ebx			  # ebx与6比较，所以一直会循环到6个数结束。
  400fe0:	75 e5                	jne    400fc7 <phase_2+0x1b>  # 
  400fe2:	eb 0c                	jmp    400ff0 <phase_2+0x44>  # 循环结束后，代码段结束。
    
  400fe4:	48 8d 6c 24 04       	lea    0x4(%rsp),%rbp          # rbp=rsp+4，所以一开始的时候，rbp指向第二个数字。
  400fe9:	bb 01 00 00 00       	mov    $0x1,%ebx               # ebx=1
  400fee:	eb d7                	jmp    400fc7 <phase_2+0x1b>   # 跳到 400fc7，开始循环
  400ff0:	48 83 c4 28          	add    $0x28,%rsp
  400ff4:	5b                   	pop    %rbx
  400ff5:	5d                   	pop    %rbp
  400ff6:	c3                   	retq   
```

注意从第一段出来的会跳转到 400fe4。详细的解析附在代码后面。此段代码要求，从第二个数字开始，会依次比前一个数字增加1，2，3，4，5。已知第一个数字为0，所以6个数字依次为 `0 1 3 6 10 15`

#### 4. <Phase_3>

**第一段**

```python
0000000000400ff7 <phase_3>:
  400ff7:	48 83 ec 18          	sub    $0x18,%rsp					 # 为在堆栈中放如数字做准备
  400ffb:	48 8d 4c 24 0c       	lea    0xc(%rsp),%rcx				 # rcx指向rsp+c
  401000:	48 8d 54 24 08       	lea    0x8(%rsp),%rdx				 # rdx指向rsp+8
  401005:	be ad 29 40 00       	mov    $0x4029ad,%esi				 # 这里面存储的是格式化形式，输入需与此一致
  40100a:	b8 00 00 00 00       	mov    $0x0,%eax
  40100f:	e8 9c fc ff ff       	callq  400cb0 <__isoc99_sscanf@plt>  # 此函数为读取函数，会改变eax的值。输入一个，eax加一个1
  401014:	83 f8 01             	cmp    $0x1,%eax                     # eax需大于1，所以需要输入两个以上数字
  401017:	7f 05                	jg     40101e <phase_3+0x27>
  401019:	e8 83 06 00 00       	callq  4016a1 <explode_bomb>
```

从内存中读取 0x4029ad 中的值，输出为 `%d %d`，说明输入的前两个需要为整型数据。而且第一个数存储在 rsp+8（rdx），第二个数存储在 rsp+c（rcx）。

**第二段**

```python
  40101e:	83 7c 24 08 07       	cmpl   $0x7,0x8(%rsp)  				 # 输入的第一个数不能大于7
  401023:	77 66                	ja     40108b <phase_3+0x94>         # 对应的是炸弹爆炸的函数
```

需保证输入的第一个数不大于7。

**第三段**

```python
  401025:	8b 44 24 08          	mov    0x8(%rsp),%eax        # eax中存入第一个数
  401029:	ff 24 c5 e0 26 40 00 	jmpq   *0x4026e0(,%rax,8)    # 跳转到以 4026e0+8*rax 中的值对应的地址
```

**第四段**

```python
  401030:	b8 00 00 00 00       	mov    $0x0,%eax
  401035:	eb 05                	jmp    40103c <phase_3+0x45>
  401037:	b8 1b 02 00 00       	mov    $0x21b,%eax       
  40103c:	2d c6 01 00 00       	sub    $0x1c6,%eax
  401041:	eb 05                	jmp    401048 <phase_3+0x51>
  401043:	b8 00 00 00 00       	mov    $0x0,%eax
  401048:	05 e6 02 00 00       	add    $0x2e6,%eax
  40104d:	eb 05                	jmp    401054 <phase_3+0x5d>
  40104f:	b8 00 00 00 00       	mov    $0x0,%eax
  401054:	2d f9 01 00 00       	sub    $0x1f9,%eax
  401059:	eb 05                	jmp    401060 <phase_3+0x69>
  40105b:	b8 00 00 00 00       	mov    $0x0,%eax
  401060:	05 f9 01 00 00       	add    $0x1f9,%eax
  401065:	eb 05                	jmp    40106c <phase_3+0x75>
  401067:	b8 00 00 00 00       	mov    $0x0,%eax
  40106c:	2d f9 01 00 00       	sub    $0x1f9,%eax
  401071:	eb 05                	jmp    401078 <phase_3+0x81>
  401073:	b8 00 00 00 00       	mov    $0x0,%eax
  401078:	05 f9 01 00 00       	add    $0x1f9,%eax  
  40107d:	eb 05                	jmp    401084 <phase_3+0x8d>
  40107f:	b8 00 00 00 00       	mov    $0x0,%eax       
  401084:	2d f9 01 00 00       	sub    $0x1f9,%eax     
  401089:	eb 0a                	jmp    401095 <phase_3+0x9e>
```

这段代码的每行的代码地址和刚才内存4026e0中的数值有关系。根据第一个数可以求出要找的内存 `4026e0+8*rax` 是多少，这个内存里面存储的数据就是将要跳转的地址，此地址在此段代码中。从跳转代码开始向下进行一系列的算数运算，得到eax的值。

**第五段**

```python
  40108b:	e8 11 06 00 00       	callq  4016a1 <explode_bomb>
  401090:	b8 00 00 00 00       	mov    $0x0,%eax
    
  401095:	83 7c 24 08 05       	cmpl   $0x5,0x8(%rsp)          # 比较第一个数字和5的大小
  40109a:	7f 06                	jg     4010a2 <phase_3+0xab>   # 要求第一个数字必须小于等于5
    
  40109c:	3b 44 24 0c          	cmp    0xc(%rsp),%eax          # 比较刚才计算出的eax和输入的第二个数字的大小
  4010a0:	74 05                	je     4010a7 <phase_3+0xb0>   # 要求二者必须相等。
  4010a2:	e8 fa 05 00 00       	callq  4016a1 <explode_bomb>
  4010a7:	48 83 c4 18          	add    $0x18,%rsp
  4010ab:	c3                   	retq  
```

从整段代码来看，要求输入的第一个数字要小于等于5，假设第一个数字为3，那么 `4026e0+8*rax` = 4026f8，根据命令 `w/s 0x4026f8`，输出 ``，跳转到第四段代码中的相应地址，向后进行运算，得到 eax=0x142，换算成十进制为322，所以其中一个答案为  3 322。事实上，此题应有6个答案，分别对应着第一个输入的数字从0到5的不同情况。

#### 5. <Phase_4>

**第一段**

```python
00000000004010ea <phase_4>:
  4010ea:	48 83 ec 18          	sub    $0x18,%rsp
  4010ee:	48 8d 4c 24 0c       	lea    0xc(%rsp),%rcx
  4010f3:	48 8d 54 24 08       	lea    0x8(%rsp),%rdx
  4010f8:	be ad 29 40 00       	mov    $0x4029ad,%esi
  4010fd:	b8 00 00 00 00       	mov    $0x0,%eax
  401102:	e8 a9 fb ff ff       	callq  400cb0 <__isoc99_sscanf@plt>
  401107:	83 f8 02             	cmp    $0x2,%eax                      # 前两个输入为数字
  40110a:	75 07                	jne    401113 <phase_4+0x29>
  40110c:	83 7c 24 08 0e       	cmpl   $0xe,0x8(%rsp)                 #
  401111:	76 05                	jbe    401118 <phase_4+0x2e>          # 输入的第一个数字要小于等于 e
  401113:	e8 89 05 00 00       	callq  4016a1 <explode_bomb>
```

此段代码与phase_3的第一段代码基本一样，要求输入的前两个为整型数字，第一个数存储在 rsp+8（rdx），第二个数存储在 rsp+c（rcx）。而且倒数2、3行要求输入的第一个数字要小于等于e。

**第二段**

```python
  401118:	ba 0e 00 00 00       	mov    $0xe,%edx				# 初始化，edx=e
  40111d:	be 00 00 00 00       	mov    $0x0,%esi				# esi=0
  401122:	8b 7c 24 08          	mov    0x8(%rsp),%edi        	# edi=输入的第一个数
  401126:	e8 81 ff ff ff       	callq  4010ac <func4>           # 调用函数，可写为 func4(edx, esi, edi)
    
  40112b:	83 f8 07             	cmp    $0x7,%eax   				# 调用结束，要求eax的值为7
  40112e:	75 07                	jne    401137 <phase_4+0x4d>	#
  401130:	83 7c 24 0c 07       	cmpl   $0x7,0xc(%rsp)			# 要求输入的第二个数字等于7
  401135:	74 05                	je     40113c <phase_4+0x52>    #
  401137:	e8 65 05 00 00       	callq  4016a1 <explode_bomb>
  40113c:	48 83 c4 18          	add    $0x18,%rsp
  401140:	c3                   	retq  
```

推断：根据调用结束时要求eax=7，应该可以应用func4反推出第一个数字。下面看调用函数。

**func4（edx，esi，edi）**

```python
00000000004010ac <func4>:
  4010ac:	48 83 ec 08          	sub    $0x8,%rsp           
  4010b0:	89 d0                	mov    %edx,%eax			# edx的值赋给eax，一开始（eax=edx=e）
  4010b2:	29 f0                	sub    %esi,%eax			# eax=eax-esi，
  4010b4:	89 c1                	mov    %eax,%ecx			# 同时将值存在ecx中
  4010b6:	c1 e9 1f             	shr    $0x1f,%ecx 		 	# ecx右移31位后，将值加到eax上。
  4010b9:	01 c8                	add    %ecx,%eax  		    # 即eax=eax+eax>>31，或者 若eax为非负数则不变，为负数则加1
  4010bb:	d1 f8                	sar    %eax       		    # eax除以2
  4010bd:	8d 0c 30             	lea    (%rax,%rsi,1),%ecx   # ecx = rax+rsi
  4010c0:	39 f9                	cmp    %edi,%ecx            
  4010c2:	7e 0c                	jle    4010d0 <func4+0x24>
  4010c4:	8d 51 ff             	lea    -0x1(%rcx),%edx      # edx=rcx-1
  4010c7:	e8 e0 ff ff ff       	callq  4010ac <func4>
  4010cc:	01 c0                	add    %eax,%eax
  4010ce:	eb 15                	jmp    4010e5 <func4+0x39>
    
  4010d0:	b8 00 00 00 00       	mov    $0x0,%eax
  4010d5:	39 f9                	cmp    %edi,%ecx
  4010d7:	7d 0c                	jge    4010e5 <func4+0x39>
  4010d9:	8d 71 01             	lea    0x1(%rcx),%esi
  4010dc:	e8 cb ff ff ff       	callq  4010ac <func4>
  4010e1:	8d 44 00 01          	lea    0x1(%rax,%rax,1),%eax
    
  4010e5:	48 83 c4 08          	add    $0x8,%rsp
  4010e9:	c3                   	retq   
```

由第 4010c7 和 4010dc 两行可知，此函数为递归函数。下面用类似于c语言的方法翻译此段代码。

```c++
func4（edx, esi, edi）{
    eax = edx;							// mov %edx, %eax
    eax -= esi;							// sub %esi, %eax
    eax = eax + eax >> 31;				// mov %eax, %ecx; shr $0x1f, %ecx; add %ecx, %eax
    eax /= 2;							// sar %eax
    ecx = eax + esi;					// lea (%rax, %rsi, 1), %ecx
    
    if (ecx <= edi){					// cmp %edi, %ecx; jle 4010d0 <func4+0x24>
        eax = 0;						// mov $0x0, %eax
        if (ecx == edi)					// cmp %edi, %ecx; jge 4010e5<>// 本来这里是大于等于，但是因为前面已经有了小于等于的限制，就简化了
            return;						// add $0x8, %rsp; retq
        else{							
            esi = ecx + 1;				// lea 0x1(%rcx), %esi
            func4(edx, esi, edi);		// callq 4010ac <func4>
            eax = 2*eax + 1;			// lea 0x1(%rax, %rax, 1), %eax
            return;						// add $0x8, %rsp; retq
        }
    }
    else{
        edx = ecx - 1;					// lea    -0x1(%rcx), %edx
        func(edx, esi, edi);			// callq  4010ac <func4>
        eax = 2*eax;					// add    %eax, %eax
        return;							// jmp    4010e5 <func4+0x39>; add $0x8, %rsp; retq
    }
}
```

根据前面的推导，要求函数输出时eax为7，观察此函数的返回点，发现调用的第一个递归函数必须在第二个return处返回，且返回值为3；根据此，调用的第二个递归函数也必须在第二个return处返回，且返回值为1；则调用的第三个递归应该在第一个return处返回，返回eax=0。为了更加详细的说明，将详细的调用过程写在下面。

```
// 主函数
eax=e/2=7
ecx=7
7<edi(为了满足进入第二个return的条件)
eax=0
esi=8  
			--->    // 调用第一个递归
					eax=e-8=6
					eax=6/2=3
					ecx=3+8=b
					b<edi(为了满足进入第二个return的条件)
					eax=0
					esi=c
								--->	// 调用第二个递归
										eax=e-c=2
										eax=2/2=1
										ecx=1+c=d
										d<edi(为了满足进入第二个return的条件)
										eax=0
										esi=e
													--->	// 调用第三个递归
															eax=e-e=0
															eax=0/2=0
															ecx=e
															e = edi(为了满足进入第一个return的条件)！！！
*返回eax=7 <<-----  *返回eax=3  <<----  *返回eax=1  <<-----   *返回eax=0
```

根据上图，第一个数字（edi）需要等于e。综上所述，输入为 14 7。

#### 6. <Phase_5>

**第一段**

```python
0000000000401141 <phase_5>:
  401141:	48 83 ec 18          	sub    $0x18,%rsp
  401145:	48 8d 4c 24 0c       	lea    0xc(%rsp),%rcx
  40114a:	48 8d 54 24 08       	lea    0x8(%rsp),%rdx
  40114f:	be ad 29 40 00       	mov    $0x4029ad,%esi
  401154:	b8 00 00 00 00       	mov    $0x0,%eax
  401159:	e8 52 fb ff ff       	callq  400cb0 <__isoc99_sscanf@plt>
  40115e:	83 f8 01             	cmp    $0x1,%eax
  401161:	7f 05                	jg     401168 <phase_5+0x27>
  401163:	e8 39 05 00 00       	callq  4016a1 <explode_bomb>
```

与前面一样，要求输入的前两个为整型数字，第一个数存储在 rsp+8（rdx），第二个数存储在 rsp+c（rcx）。

**第二段**

```python
  401168:	8b 44 24 08          	mov    0x8(%rsp),%eax         # 第一个数字存在eax中
  40116c:	83 e0 0f             	and    $0xf,%eax			  # eax = eax & 0xf
  40116f:	89 44 24 08          	mov    %eax,0x8(%rsp)		  #
  401173:	83 f8 0f             	cmp    $0xf,%eax			  # 
  401176:	74 2c                	je     4011a4 <phase_5+0x63>  # 这一段要求输入的第一个数的最低位不能为f
    
  401178:	b9 00 00 00 00       	mov    $0x0,%ecx			  # 循环的初始化，ecx=0，ecx的作用相当于求所有eax的和
  40117d:	ba 00 00 00 00       	mov    $0x0,%edx			  # edx=0
    
  401182:	83 c2 01             	add    $0x1,%edx			  # edx+1
  401185:	48 98                	cltq   						  # 字符扩展，不影响运算结果。
  401187:	8b 04 85 20 27 40 00 	mov    0x402720(,%rax,4),%eax # eax =（402720+4*rax）；此地址中的值
  40118e:	01 c1                	add    %eax,%ecx			  # ecx+=eax 
  401190:	83 f8 0f             	cmp    $0xf,%eax			  # eax与f比较，不相等的话返回 401182 继续循环
  401193:	75 ed                	jne    401182 <phase_5+0x41>  # 
    
  401195:	89 44 24 08          	mov    %eax,0x8(%rsp)		  # eax=第一个数
  401199:	83 fa 0f             	cmp    $0xf,%edx			  # edx与f比较，要求必须相等。从上面的循环可知，每循环一次，edx会加1，所以
  40119c:	75 06                	jne    4011a4 <phase_5+0x63>  # 必须要循环15次，也就是前14次从内存读出的值都不是f，直到第十五次读出f。
    
  40119e:	3b 4c 24 0c          	cmp    0xc(%rsp),%ecx		  # 比较第二个数与ecx（sum和），要求相等。
  4011a2:	74 05                	je     4011a9 <phase_5+0x68>
    
  4011a4:	e8 f8 04 00 00       	callq  4016a1 <explode_bomb>
  4011a9:	48 83 c4 18          	add    $0x18,%rsp
  4011ad:	c3                   	retq
```

然后可以从 0x402720 内存开始打印信息，如下所示。每个地址对应着rax从0开始依次加1，可以对应的写在后面。

```
0x402720 <array.3495>:  10				# 0
(gdb) x/wd 0x402724
0x402724 <array.3495+4>:        2		# 1
(gdb) x/wd 0x402728
0x402728 <array.3495+8>:        14		# 2
(gdb) x/wd 0x40272c
0x40272c <array.3495+12>:       7		# 3
(gdb) x/wd 0x402730
0x402730 <array.3495+16>:       8		# 4
(gdb) x/wd 0x402734
0x402734 <array.3495+20>:       12		# 5
(gdb) x/wd 0x402738
0x402738 <array.3495+24>:       15		# 6
(gdb) x/wd 0x40273c
0x40273c <array.3495+28>:       11		# 7
(gdb) x/wd 0x402740
0x402740 <array.3495+32>:       0		# 8
(gdb) x/wd 0x402744
0x402744 <array.3495+36>:       4		# 9
(gdb) x/wd 0x402748
0x402748 <array.3495+40>:       1		# 10
(gdb) x/wd 0x40274c
0x40274c <array.3495+44>:       13		# 11
(gdb) x/wd 0x402750
0x402750 <array.3495+48>:       3		# 12
(gdb) x/wd 0x402754
0x402754 <array.3495+52>:       9		# 13
(gdb) x/wd 0x402758
0x402758 <array.3495+56>:       6		# 14
(gdb) x/wd 0x40275c
0x40275c <array.3495+60>:       5		# 15

从前面的分析可知，最后一次读取的应为 eax=f，对应上一个eax为6 -> 14 -> 2 -> 1 -> 10 -> 0 -> 8 -> 4 -> 9 -> 13 -> 11 -> 7 -> 3 -> 12 -> 5所以输入的第一个数字应为5。
ecx是所有eax的求和，但是一开始的5并没有加进去（第一个加的数是5对应的地址存的12），所以最后的ecx为 1+2+..+16 - 5 = 115
```

#### 7. <Phase_6>

**第一段**

```
00000000004011ae <phase_6>:
  4011ae:	41 55                	push   %r13
  4011b0:	41 54                	push   %r12
  4011b2:	55                   	push   %rbp
  4011b3:	53                   	push   %rbx
  4011b4:	48 83 ec 58          	sub    $0x58,%rsp
  4011b8:	48 89 e6             	mov    %rsp,%rsi        # rsi指向第一个数字的地址
  4011bb:	e8 17 05 00 00       	callq  4016d7 <read_six_numbers>
```

又后三行以及函数`read_six_numbers`可知，此段代码会读入六个数字。假设从低地址到高地址分别为m1, m2, m3, m4, m5, m6。

**第二段**

> ```
>   4011c0:	49 89 e5             	mov    %rsp,%r13         # 初始化，将 r13 指向第一个数字    
>   4011c3:	41 bc 00 00 00 00    	mov    $0x0,%r12d        # 外循环的计数器 r12 初始化为0
> 
>   4011c9:	4c 89 ed             	mov    %r13,%rbp  # 外循环从此行开始，`mov %r13,%rbp` 保证基数一直存储在rbp里面
>   
>   4011cc:	41 8b 45 00          	mov    0x0(%r13),%eax          # 4-8行：取出 r13 中的数字，存入 eax 中，
>   4011d0:	83 e8 01             	sub    $0x1,%eax               # 而且要保证这个数字小于等于6。
>   4011d3:	83 f8 05             	cmp    $0x5,%eax               #
>   4011d6:	76 05                	jbe    4011dd <phase_6+0x2f>   #
>   4011d8:	e8 c4 04 00 00       	callq  4016a1 <explode_bomb>   #
> 
>   4011dd:	41 83 c4 01          	add    $0x1,%r12d              # 外循环的计数器 r12 加 1
>   4011e1:	41 83 fc 06          	cmp    $0x6,%r12d              # 不等于6则跳转到 4011ee，
>   4011e5:	75 07                	jne    4011ee <phase_6+0x40>   # 相等则令 esi=0，并跳出外循环。
>   4011e7:	be 00 00 00 00       	mov    $0x0,%esi               # 说明外面的循环体要执行6次，即遍历
>   4011ec:	eb 42                	jmp    401230 <phase_6+0x82>   # 输入的6个数字。
> 
>   4011ee:	44 89 e3             	mov    %r12d,%ebx              # r12 存储的是外循环的循环次数，第一次循环时 r12=1，后面依次累加。
>                                                                    # 此步相当于小循环计数器 ebx 的初始化。
>   4011f1:	48 63 c3             	movslq %ebx,%rax               # ebp是大循环中基数的指针，eax是小循环中的指针，每次都从基数的后一个数
>   4011f4:	8b 04 84             	mov    (%rsp,%rax,4),%eax      # 开始遍历。
>   4011f7:	39 45 00             	cmp    %eax,0x0(%rbp)          # 由引爆规则，ebp!=eax
>   4011fa:	75 05                	jne    401201 <phase_6+0x53>   #
>   4011fc:	e8 a0 04 00 00       	callq  4016a1 <explode_bomb>   #
> 
>   401201:	83 c3 01             	add    $0x1,%ebx               # 小循环指针后移
>   401204:	83 fb 05             	cmp    $0x5,%ebx               # 若ebp还没有遍历完最后一个数字
>   401207:	7e e8                	jle    4011f1 <phase_6+0x43>   # 返回 4011f1 继续
>   401209:	49 83 c5 04          	add    $0x4,%r13               # 若遍历结束，则将 r13 指向下一个数字作为新的基数
>   40120d:	eb ba                	jmp    4011c9 <phase_6+0x1b>   # 并进入一个新的小循环。
> ```
>
> 

这段代码有两个循环。外循环从第一个数开始遍历（每个数相当于基数），内循环从当前的基数向后遍历。详细的代码解释附在代码后面。此段代码的作用是要求6个数字每个数字必须小于等于6，而且两两不相等。

**第三段**

```python
  40120f:	48 8b 52 08          	mov    0x8(%rdx),%rdx          # 数字大于1的情况：
  401213:	83 c0 01             	add    $0x1,%eax               # eax一直加1直到与ecx中的数字相等，
  401216:	39 c8                	cmp    %ecx,%eax               # 每次加1，edx地址加8再作为地址取值，放入edx
  401218:	75 f5                	jne    40120f <phase_6+0x61>   #
  40121a:	eb 05                	jmp    401221 <phase_6+0x73>   #
    
  40121c:	ba 10 43 60 00       	mov    $0x604310,%edx           # 数字小于等于1的情况：edx=0x604310
                                                                    
  401221:	48 89 54 74 20       	mov    %rdx,0x20(%rsp,%rsi,2)   # 为了更好的说明，让 A=rsp+20，由于rsi是从0开始依次加4
  401226:	48 83 c6 04          	add    $0x4,%rsi                # 所以从地址A开始的栈中依次存放着每个mi(i=1...6)对应的edx的值
  40122a:	48 83 fe 18          	cmp    $0x18,%rsi               # 
  40122e:	74 14                	je     401244 <phase_6+0x96>    # 若所有的数字都遍历完了，跳出第三段的循环
    
  401230:	8b 0c 34             	mov    (%rsp,%rsi,1),%ecx       # 初始时 esi=0。由 401226 行的代码可知，每次rsi加4，所以ecx中存的
  401233:	83 f9 01             	cmp    $0x1,%ecx                # 是每次循环对应的数字。
  401236:	7e e4                	jle    40121c <phase_6+0x6e>    # 以第一个数字 m1 为例。
  401238:	b8 01 00 00 00       	mov    $0x1,%eax                # 若 m1 小于等于 1，跳到 40121c
  40123d:	ba 10 43 60 00       	mov    $0x604310,%edx           # 若大于 1，初始化：eax=1，edx=0x604310，跳到40120f
  401242:	eb cb                	jmp    40120f <phase_6+0x61>    # 具体的内容看上面的对应代码块
```

第二段出外循环后会跳转到 0x401230 处，所以从401230处开始分析。注意第二段跳出大循环之前 esi=0。其中 0x604310 一直作为地址使用，查看其内容：

```
(gdb) x/12a 0x604310
0x604310 <node1>:       0x100000352     0x604320 <node2>
0x604320 <node2>:       0x200000133     0x604330 <node3>
0x604330 <node3>:       0x300000204     0x604340 <node4>
0x604340 <node4>:       0x400000213     0x604350 <node5>
0x604350 <node5>:       0x50000018b     0x604360 <node6>
0x604360 <node6>:       0x6000000fc     0x0
```

这类似于一个链表，其中每一行类似于一个结构体，第一个元素是一个整数值，第二个元素是下一个结构的首地址。所以 401221 对应的语句中，每次都是将 edx 指向了下一个结构体。所以本代码段的含义为，如果第 i 个数字为1，则将604310存入（rsp+20+8i）中；如果第 i 个数字为m（m!=1)，则将 6043m0存入（rsp+20+8i）中。例如，输入 2 3 4 1 5 6，则从rsp+20开始的单元中，依次写入 604320，604330，604340，604310，604350，604360。

**第四段**

```python
  401244:	48 8b 5c 24 20       	mov    0x20(%rsp),%rbx       # rsp+20 开始存储的是上面所述的地址信息，所以一开始rbx中存第一个地址
  401249:	48 8d 44 24 28       	lea    0x28(%rsp),%rax       # rax = rsp+28
  40124e:	48 8d 74 24 50       	lea    0x50(%rsp),%rsi       # rsi = rsp+50，这是该段的最后一个数字对应的地址
  401253:	48 89 d9             	mov    %rbx,%rcx             # 若rcx存第i个地址
    
  401256:	48 8b 10             	mov    (%rax),%rdx           # 则rdx存第i+1个地址
  401259:	48 89 51 08          	mov    %rdx,0x8(%rcx)        # 从上面可知，rcx+8这个地址对应的是链表中该节点连接的下一节点的地址信息
                                                                 # 将rdx的值赋给这个地址，就说明此时第i个节点的后续节点是第i+1个节点，而不
                                                                 # 是以前按正常顺序604310，604320，30，40，50，60。
  40125d:	48 83 c0 08          	add    $0x8,%rax             # rax在以前的基础上加8，指向了第i+2个地址
  401261:	48 39 f0             	cmp    %rsi,%rax             # 比较rax和rsi的大小，相等才结束此段代码。说明rax要一直遍历完6个数字。
  401264:	74 05                	je     40126b <phase_6+0xbd> # 
  401266:	48 89 d1             	mov    %rdx,%rcx             # 一直不相等的话，rcx中存储rdx的值，这时候rcx存的就是第i+1个地址。
  401269:	eb eb                	jmp    401256 <phase_6+0xa8> # 因为操作的是rdx和rcx，这时候再返回去rdx就会存储第i+2个地址。
  40126b:	48 c7 42 08 00 00 00 	movq   $0x0,0x8(%rdx)        # 令链表最后一项的next指针指向 0
  401272:	00 
```

此段代码的作用就是让存储空间中以604310为地址开始的链表的连接顺序与我们输入的数字的顺序相等。仍以上面的顺序为例，此时链表会从604320进入（第二个节点），后面依次连接604330，604340，604310，604350，604360代表的节点。

**第五段**

```python
  401273:	bd 05 00 00 00       	mov    $0x5,%ebp             # ebp=5，循环次数计数器
        
  401278:	48 8b 43 08          	mov    0x8(%rbx),%rax        # 在第四段rbx已经存储了第一个数对应的节点的地址，rax存储其后继节点的地址
  40127c:	8b 00                	mov    (%rax),%eax           # eax存的是后继节点的第一个元素的值（在打印信息中，如352，133等）
  40127e:	39 03                	cmp    %eax,(%rbx)           # rbx是本节点的第一个元素的值
  401280:	7e 05                	jle    401287 <phase_6+0xd9> # 必须要求本节点的元素值小于其后继节点的元素值
  401282:	e8 1a 04 00 00       	callq  4016a1 <explode_bomb>
  401287:	48 8b 5b 08          	mov    0x8(%rbx),%rbx        # rbx指向本节点的后继节点
  40128b:	83 ed 01             	sub    $0x1,%ebp             # 计数器ebp-1
  40128e:	75 e8                	jne    401278 <phase_6+0xca> # ebp 不为1则一直循环前后节点比较的操作，为1则跳出
  401290:	48 83 c4 58          	add    $0x58,%rsp            # 后面都是堆栈返回的操作
  401294:	5b                   	pop    %rbx
  401295:	5d                   	pop    %rbp
  401296:	41 5c                	pop    %r12
  401298:	41 5d                	pop    %r13
  40129a:	c3                   	retq   
```

此段代码要求，前一个节点的元素值必须小于后一个节点的元素值，所以节点的元素值按从小到大排列。根据上面打印的数据，应该是`0fc, 133, 18b, 204, 213, 352`，那么对应的数字的顺序就是`6 2 5 3 4 1`

#### 8. <Secret_phase>

**进入Secret_phase：phase_defused**

```python 
000000000040183f <phase_defused>:
  40183f:	48 83 ec 78          	sub    $0x78,%rsp
  401843:	64 48 8b 04 25 28 00 	mov    %fs:0x28,%rax
  40184a:	00 00 
  40184c:	48 89 44 24 68       	mov    %rax,0x68(%rsp)
  401851:	31 c0                	xor    %eax,%eax
  401853:	bf 01 00 00 00       	mov    $0x1,%edi
  401858:	e8 38 fd ff ff       	callq  401595 <send_msg>
  40185d:	83 3d 58 2f 20 00 06 	cmpl   $0x6,0x202f58(%rip)           # 6047bc <num_input_strings>
  401864:	75 6d                	jne    4018d3 <phase_defused+0x94>   # 
    
  401866:	4c 8d 44 24 10       	lea    0x10(%rsp),%r8
  40186b:	48 8d 4c 24 0c       	lea    0xc(%rsp),%rcx
  401870:	48 8d 54 24 08       	lea    0x8(%rsp),%rdx
  401875:	be f7 29 40 00       	mov    $0x4029f7,%esi                # 需要输入 %d %d %s
  40187a:	bf d0 48 60 00       	mov    $0x6048d0,%edi  				 # 查看6048d0中的值，发现是 14 7，说明是phase_4输入的位置
  40187f:	b8 00 00 00 00       	mov    $0x0,%eax
  401884:	e8 27 f4 ff ff       	callq  400cb0 <__isoc99_sscanf@plt>  #
  401889:	83 f8 03             	cmp    $0x3,%eax					 #
  40188c:	75 31                	jne    4018bf <phase_defused+0x80>   # 要求输入三个量，否则跳过<secret_phase>。从上面的分析可知
    																	 # 已经有了两个整形数据，还需要一个字符串。
    
  40188e:	be 00 2a 40 00       	mov    $0x402a00,%esi				 # 根据 phase_1 的解答，可知这一小段要求
  401893:	48 8d 7c 24 10       	lea    0x10(%rsp),%rdi				 # 输入的字符串与 0x402a00 中的相同。x/s 打印后是RrEvil。
  401898:	e8 2b fb ff ff       	callq  4013c8 <strings_not_equal>    #
  40189d:	85 c0                	test   %eax,%eax					 #
  40189f:	75 1e                	jne    4018bf <phase_defused+0x80>   #
    
  4018a1:	bf 58 28 40 00       	mov    $0x402858,%edi
  4018a6:	e8 15 f3 ff ff       	callq  400bc0 <puts@plt>
  4018ab:	bf 80 28 40 00       	mov    $0x402880,%edi
  4018b0:	e8 0b f3 ff ff       	callq  400bc0 <puts@plt>
  4018b5:	b8 00 00 00 00       	mov    $0x0,%eax
  4018ba:	e8 1a fa ff ff       	callq  4012d9 <secret_phase>         # !!!!!!!!!
    
  4018bf:	bf b8 28 40 00       	mov    $0x4028b8,%edi
  4018c4:	e8 f7 f2 ff ff       	callq  400bc0 <puts@plt>
  4018c9:	bf e8 28 40 00       	mov    $0x4028e8,%edi
  4018ce:	e8 ed f2 ff ff       	callq  400bc0 <puts@plt>
  4018d3:	48 8b 44 24 68       	mov    0x68(%rsp),%rax
  4018d8:	64 48 33 04 25 28 00 	xor    %fs:0x28,%rax
  4018df:	00 00 
  4018e1:	74 05                	je     4018e8 <phase_defused+0xa9>
  4018e3:	e8 f8 f2 ff ff       	callq  400be0 <__stack_chk_fail@plt>
  4018e8:	48 83 c4 78          	add    $0x78,%rsp
  4018ec:	c3                   	retq   
  4018ed:	0f 1f 00             	nopl   (%rax)
```

经上述代码分析可知，若想进入secret_phase，phase_4 的答案就不能只写 14 7，而要写 14 7 DrEvil。

**Secret_phase**

```python
00000000004012d9 <secret_phase>:
  4012d9:	53                   	push   %rbx
  4012da:	e8 3a 04 00 00       	callq  401719 <read_line>		    # 调用此函数读取数据，存入eax中。
  4012df:	ba 0a 00 00 00       	mov    $0xa,%edx
  4012e4:	be 00 00 00 00       	mov    $0x0,%esi
  4012e9:	48 89 c7             	mov    %rax,%rdi
  4012ec:	e8 9f f9 ff ff       	callq  400c90 <strtol@plt>
    
  4012f1:	48 89 c3             	mov    %rax,%rbx                    # 输入的数字存储在 rbx里面
  4012f4:	8d 40 ff             	lea    -0x1(%rax),%eax				# eax -= 1
  4012f7:	3d e8 03 00 00       	cmp    $0x3e8,%eax     				# 
  4012fc:	76 05                	jbe    401303 <secret_phase+0x2a>   #
  4012fe:	e8 9e 03 00 00       	callq  4016a1 <explode_bomb>        # 输入的数字要求小于等于 1000+1=1001
    
  401303:	89 de                	mov    %ebx,%esi      			    # esi 存输入的数字
  401305:	bf 30 41 60 00       	mov    $0x604130,%edi				# edi = 0x604130，估计又要找这个地址对应的内存的内容了。
  40130a:	e8 8c ff ff ff       	callq  40129b <fun7> 			    # 调用 fun7（esi, edi）
    
  40130f:	83 f8 07             	cmp    $0x7,%eax      			    # 调用fun7结束后，eax的值必须等于 7
  401312:	74 05                	je     401319 <secret_phase+0x40>   #
  401314:	e8 88 03 00 00       	callq  4016a1 <explode_bomb>		#
    
  401319:	bf b8 26 40 00       	mov    $0x4026b8,%edi
  40131e:	e8 9d f8 ff ff       	callq  400bc0 <puts@plt>
  401323:	e8 17 05 00 00       	callq  40183f <phase_defused>
  401328:	5b                   	pop    %rbx
  401329:	c3                   	retq   
  40132a:	66 0f 1f 44 00 00    	nopw   0x0(%rax,%rax,1)
```

类似于 phase_4，利用调用函数反向推测输入值。下面研究 fun7。

**fun7（esi，edi）**

```python
000000000040129b <fun7>:
  40129b:	48 83 ec 08          	sub    $0x8,%rsp
  40129f:	48 85 ff             	test   %rdi,%rdi
  4012a2:	74 2b                	je     4012cf <fun7+0x34>     
  4012a4:	8b 17                	mov    (%rdi),%edx
  4012a6:	39 f2                	cmp    %esi,%edx
  4012a8:	7e 0d                	jle    4012b7 <fun7+0x1c>
    
  4012aa:	48 8b 7f 08          	mov    0x8(%rdi),%rdi
  4012ae:	e8 e8 ff ff ff       	callq  40129b <fun7>
  4012b3:	01 c0                	add    %eax,%eax
  4012b5:	eb 1d                	jmp    4012d4 <fun7+0x39>
    
  4012b7:	b8 00 00 00 00       	mov    $0x0,%eax
  4012bc:	39 f2                	cmp    %esi,%edx
  4012be:	74 14                	je     4012d4 <fun7+0x39>
    
  4012c0:	48 8b 7f 10          	mov    0x10(%rdi),%rdi
  4012c4:	e8 d2 ff ff ff       	callq  40129b <fun7>
  4012c9:	8d 44 00 01          	lea    0x1(%rax,%rax,1),%eax
  4012cd:	eb 05                	jmp    4012d4 <fun7+0x39>
  4012cf:	b8 ff ff ff ff       	mov    $0xffffffff,%eax           # eax = -1
    
  4012d4:	48 83 c4 08          	add    $0x8,%rsp				  # return
  4012d8:	c3                   	retq   
```

比较简单，直接将此段代码写成c格式并将对应代码写在后面：(注意esi存储的是输入的数字)

```c
fun(esi, edi){
    edx = *edi;							// mov (%rdi), %edx
    
    if (edx <= esi){					// cmp %esi, %edx; jle 4012b7 <fun7+0x1c>
        eax = 0;						// mov $0x0, %eax
        if (edx == esi)					// cmp %esi, %edx; 
            return;						// je 4012d4 <fun7+0x39>
        else{
            edi = *(edi + 10);			// mov 0x10(%rdi), %rdi
            fun(esi, edi);				// callq 40129b <fun7>
            eax = 2*eax +1;				// lea 0x1(%rax, %rax, 1), %eax
            return;						// jmp 4012d4 <fun7+0x39>
        }
    }
    else{
        edi = edi+8;					// mov 0x8(%rdi), %rdi
        fun(esi, edi);					// callq 40129b <fun7>
        eax = 2*eax;					// add %eax, %eax
        return;							// 4012d4 <fun7+0x39>
    }
}
```

要求最后返回时 eax=7，和 phase_4 的思路是一致的。但是里面涉及到与 edi 有关的内存，在 secret_phase 代码段中，edi=0x604130，打印其后面多个内容如下。

```
(gdb) x/128xg 0x604130
0x604130 <n1>:  0x0000000000000024      0x0000000000604150
0x604140 <n1+16>:       0x0000000000604170      0x0000000000000000
0x604150 <n21>: 0x0000000000000008      0x00000000006041d0
0x604160 <n21+16>:      0x0000000000604190      0x0000000000000000
0x604170 <n22>: 0x0000000000000032      0x00000000006041b0
0x604180 <n22+16>:      0x00000000006041f0      0x0000000000000000
0x604190 <n32>: 0x0000000000000016      0x00000000006042b0
0x6041a0 <n32+16>:      0x0000000000604270      0x0000000000000000
0x6041b0 <n33>: 0x000000000000002d      0x0000000000604210
0x6041c0 <n33+16>:      0x00000000006042d0      0x0000000000000000
0x6041d0 <n31>: 0x0000000000000006      0x0000000000604230
0x6041e0 <n31+16>:      0x0000000000604290      0x0000000000000000
0x6041f0 <n34>: 0x000000000000006b      0x0000000000604250
0x604200 <n34+16>:      0x00000000006042f0      0x0000000000000000
0x604210 <n45>: 0x0000000000000028      0x0000000000000000
0x604220 <n45+16>:      0x0000000000000000      0x0000000000000000
0x604230 <n41>: 0x0000000000000001      0x0000000000000000
0x604240 <n41+16>:      0x0000000000000000      0x0000000000000000
0x604250 <n47>: 0x0000000000000063      0x0000000000000000
0x604260 <n47+16>:      0x0000000000000000      0x0000000000000000
0x604270 <n44>: 0x0000000000000023      0x0000000000000000
0x604280 <n44+16>:      0x0000000000000000      0x0000000000000000
0x604290 <n42>: 0x0000000000000007      0x0000000000000000
---Type <return> to continue, or q <return> to quit---return
0x6042a0 <n42+16>:      0x0000000000000000      0x0000000000000000
0x6042b0 <n43>: 0x0000000000000014      0x0000000000000000
0x6042c0 <n43+16>:      0x0000000000000000      0x0000000000000000
0x6042d0 <n46>: 0x000000000000002f      0x0000000000000000
0x6042e0 <n46+16>:      0x0000000000000000      0x0000000000000000
0x6042f0 <n48>: 0x00000000000003e9      0x0000000000000000
```

结合次内存内容与 phase_4 的思路，可以画出流程表。

```
// 主函数
edx=(edi)=(0x604130)=0x24
0x24<edi(为了满足进入第二个return的条件)
edi=(edi+10)=(0x604140)=0x604170

			--->    // 调用第一个递归
					edx=(edi)=(0x604170)=0x32
					0x32<edi(为了满足进入第二个return的条件)
					edi=(edi+10)=(0x604180)=0x6041f0
					
								--->	// 调用第二个递归
										edx=(edi)=(0x6041f0)=0x6b
										0x6b<edi(为了满足进入第二个return的条件)
										edi=(edi+10)=(0x604200)=0x6042f0
										
													--->	// 调用第三个递归
															edx=(edi)=(0x6042f0)=0x3e9
															0x3e9 = edi(为了满足进入第一个return的条件)！！！
*返回eax=7 <<-----  *返回eax=3  <<----  *返回eax=1  <<-----   *返回eax=0
```

经过上述分析，可知输入的数字应为 0x3e9 = 1001。所以最后输入1001，结束了！！！终于写完了。。。

```


```

