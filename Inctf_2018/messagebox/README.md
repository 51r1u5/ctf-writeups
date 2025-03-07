### messagebox (Attack and Defense)
---
<p>messagebox is a tcp server which allows us to read and save files in directory named data.</p>
<p>service is  running on port <code>5050</code></p>

<p>Actually, it asks username and read/write the file with that username.</p>

<pre>
<code>
$ ./message
Enter username: s1r1u5
=======Menu======
1. Save message
2. View message
Your choice: 1
Enter size (<50): 30
Enter input: Hello World
Input saved successfully.
</code>
</pre>
<pre>
<code>
$ ./message
Enter username: s1r1u5
=======Menu======
1. Save message
2. View message
Your choice: 2
Hello World
</code>
</pre>
<p style="font-color:red">Let us start our search for finding the vulnerabilities... </p>
<p> Oh wait! Do we have to reverse engineer the binary? :flushed: No, we got the source code there. :relieved: Now our only task is to find the vulnerabilities in the source code <a href="message.c">message.c</a>.</p>

<pre><code>
void readfile(char* filename)
{
  char* ptr;
  char* temp="cat ";
  ptr=(char*)malloc(sizeof(char)*(strlen(temp)+strlen(filename)+1));
  strcpy(ptr, temp);
  strcat(ptr, filename);
  system(ptr);
  free(ptr);
  printf("\n");
}
</pre></code>
<p> Are you able to find any vulnerabilities there????</p>
<p> Yes, there it is <b>cat</b> and <b>system</b> call</p>
<p> To read the file called filename from the local directory it is using cat command. Here filename is nothing but the username which we have given.</p>

<h3>Bug 1</h3>
username = *
<p>
exploit : <a href="exp.py">exp.py</a></p>
<p> what if we give the username as *. It just read all the files in the current directory</p>
<hr/>
<h3>Bug 2</h3>
username = *; sh
<p>
exploit : <a href="exp3.py">exp3.py</a></p>
<pre>
<code>
cat *; sh
</code>
</pre>
<p> we can get the shell by ending first command by <b>;</b></p>
<p> Patch for above bugs : Rather than using dangerous system call, we can just read the files in the directory by file handling in C language.
<hr/>
<h3>Bug 3</h3>
<pre><code>
void view(char* user)
{
  int len;
  int flag=0;
  listdir();
  for(int i=0; i < count ;i++)
  {
    len=strlen(user);
    if(!strncmp(user, files[i], len))
    {
      readfile(files[i]);
      flag=1;
    }
  }
  if(!flag)
    puts("No such user");
}
</code></pre>
<p> Another bug if we give empty username as input then strncmp("",files[i],0) is always 0. </p>
<p>exploit : <a href="exp2.py">exp2.py</a></p>
<p> Patch : Simple patch can be user input must be some length.</p>
<hr/>
<p>Here is the final bug and my favorite one</p>
<h3>Bug 4 (Bufferoverflow)</h3>
<pre><code>

void save(char* user)
{
&nbsp;&nbsp;listdir(); //storing the list of files in the data directory in the global variable named files

&nbsp;&nbsp;char* temp=(char*)malloc(50*sizeof(char)); //allocating 50 bytes on the heap.
&nbsp;&nbsp;bool* exist=(bool*)malloc(count*sizeof(bool)); //allocating n bytes on the heap, here n is number of files in the directory.

&nbsp;&nbsp;int size=0;

&nbsp;&nbsp;if(!check_existence(user,exist))
&nbsp;&nbsp;{
&nbsp;&nbsp;&nbsp;&nbsp;FILE *fp;
&nbsp;&nbsp;&nbsp;&nbsp;fp=fopen(user, "w");
&nbsp;&nbsp;&nbsp;&nbsp;printf("Enter size ( < 50): "); //wtf 
&nbsp;&nbsp;&nbsp;&nbsp;size=getint();
&nbsp;&nbsp;&nbsp;&nbsp;if(size > = 50) // we need to somehow bypass this constraint 
&nbsp;&nbsp;&nbsp;&nbsp;{
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;puts("Sorry, Your data is too big!");
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;exit(0);
&nbsp;&nbsp;&nbsp;&nbsp;}
&nbsp;&nbsp;&nbsp;&nbsp;printf("Enter input: ");
&nbsp;&nbsp;&nbsp;&nbsp;get_inp(temp, size);
&nbsp;&nbsp;&nbsp;&nbsp;fprintf(fp, "%s", temp);
&nbsp;&nbsp;&nbsp;&nbsp;fclose(fp);
&nbsp;&nbsp;&nbsp;&nbsp;puts("Input saved successfully.");
&nbsp;&nbsp;}

&nbsp;&nbsp;else
&nbsp;&nbsp;{
&nbsp;&nbsp;&nbsp;&nbsp;puts("save: Permission denied. You can only view :- ");
&nbsp;&nbsp;}

&nbsp;&nbsp;for(int i=0;i < count;i++)
&nbsp;&nbsp;{
&nbsp;&nbsp;&nbsp;&nbsp;if(exist[i])  //change the value of exist[i] other than 0, so we can read the files in the directory. 
&nbsp;&nbsp;&nbsp;&nbsp;{
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;readfile(files[i]);
&nbsp;&nbsp;&nbsp;&nbsp;}
&nbsp;&nbsp;}

&nbsp;&nbsp;return;
}


</code></pre>
<p> If you look at keenly, the allocation of <b>exist(bool)</b> is just after the allocation of <b>temp</b>. So overflowing temp, we can overwrite the exist array with any value other than 0.</p>
<p>There is a constrain that the size of temp must be < 50. So we can bypass this by giving size = -1 (0xffffffff).</p>
<p> Now we can use pwn tools to do the automation stuff.</p>
<p> exploit : <a href = "exp4.py">exp4.py</a></p>
 <p>Patch: It is just enough to change unsigned int to int in the getint function.</p>
 
 :relieved: 
 
 
  
