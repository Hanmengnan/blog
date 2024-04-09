title: Linux 简单文件系统实现
tags:
  - 操作系统
  - C++
categories:
  - 笔记
author: siegelion
date: 2019-12-08 17:51:00
---

---
## Linux 简单文件系统实现

<!--与实际Linux的文件系统的实现方法存在差异，实现的功能也很少，只是从自己的理解层面和实现的难易角度出发，实现了一个简单的系统-->

### 前言

[借鉴于Linux文件系统的实现](https://www.cnblogs.com/pipci/p/10179502.html) 

#### 存储设备分区

文件系统的最终目的是把大量数据有组织的放入持久性的存储设备中，比如硬盘和磁盘。这些存储设备与内存不同。它们的存储能力具有持久性，不会因为断电而消失；存储量大，但读取速度慢。



观察常见存储设备。最开始的区域是MBR，用于Linux开机启动(参考[Linux开机启动](http://www.cnblogs.com/vamei/archive/2012/09/05/2672039.html))。剩余的空间可能分成数个分区(partition)。每个分区有一个相关的分区表(Partition table)，记录分区的相关信息。这个分区表是储存在分区之外的。分区表说明了对应分区的起始位置和分区的大小。

![img](https://images0.cnblogs.com/blog/413416/201402/251719082508830.png)

 

我们在Windows系统常常看到C分区、D分区等。Linux系统下也可以有多个分区，但都被挂载在同一个文件系统树上。

数据被存入到某个分区中。一个典型的Linux分区包含有下面各个部分:

![img](https://images0.cnblogs.com/blog/413416/201402/250221581092754.png)

 

分区的第一个部分是启动区(Boot block)，它主要是为计算机开机服务的。[Linux开机启动](http://www.cnblogs.com/vamei/archive/2012/09/05/2672039.html)后，会首先载入MBR，随后MBR从某个硬盘的启动区加载程序。该程序负责进一步的操作系统的加载和启动。为了方便管理，即使某个分区中没有安装操作系统，Linux也会在该分区预留启动区。

启动区之后的是超级区(Super block)。它存储有文件系统的相关信息，包括文件系统的类型，inode的数目，数据块的数目。

随后是多个inodes，它们是实现文件存储的关键。在Linux系统中，一个文件可以分成几个数据块存储，就好像是分散在各地的龙珠一样。为了顺利的收集齐龙珠，我们需要一个“雷达”的指引：该文件对应的inode。每个文件对应一个inode。这个inode中包含多个指针，指向属于该文件各个数据块。当操作系统需要读取文件时，只需要对应inode的"地图"，收集起分散的数据块，就可以收获我们的文件了。

#### inode简介

正如上一节中提到的，inode储存由一些指针，这些指针指向存储设备中的一些数据块，文件的内容就储存在这些数据块中。当Linux想要打开一个文件时，只需要找到文件对应的inode，然后沿着指针，将所有的数据块收集起来，就可以在内存中组成一个文件的数据了。

![img](https://images0.cnblogs.com/blog/413416/201402/251115315292334.png)

 															

### inode示例 

在Linux中，我们通过解析路径，根据沿途的目录文件来找到某个文件。目录中的条目除了所包含的文件名，还有对应的inode编号。当我们输入$cat /var/test.txt时，Linux将在根目录文件中找到var这个目录文件的inode编号，然后根据inode合成var的数据。随后，根据var中的记录，找到text.txt的inode编号，沿着inode中的指针，收集数据块，合成text.txt的数据。整个过程中，我们参考了三个inode：根目录文件，var目录文件，text.txt文件的inodes。

在Linux下，可以使用$stat filename，来查询某个文件对应的inode编号。

​                                               ![img](https://images0.cnblogs.com/blog/413416/201402/251216497524884.png)

在存储设备中实际上存储为：

![img](https://images0.cnblogs.com/blog/413416/201402/252007477914143.png)

 

当我们读取一个文件时，实际上是在目录中找到了这个文件的inode编号，然后根据inode的指针，把数据块组合起来，放入内存供进一步的处理。当我们写入一个文件时，是分配一个空白inode给该文件，将其inode编号记入该文件所属的目录，然后选取空白的数据块，让inode的指针指像这些数据块，并放入内存中的数据。



### 文件系统设计

#### 运行效果

![5](F:\HP\文档\课件\操作系统\Linux 简单文件系统实现.assets\5.png)

​                                    ![6](F:\HP\文档\课件\操作系统\Linux 简单文件系统实现.assets\6.png) 

#### 系统层次


![image-20191208161446285](C:\Users\HP\AppData\Roaming\Typora\typora-user-images\image-20191208161446285.png)

如图所示，文件系统的主要数据结构为“森林”，由于Linux系统没有盘符的概念，默认的首层目录即存在文件，故森林中的每一棵树为首层目录下的文件夹及文件。并且每棵树之间靠**兄弟指针**相互关联，但此处的指针为单向指针，即由该层中的**第一个节点**（文件/文件夹）出发，依次指向该层中后一个节点（文件/文件夹），也即第一个指向第二个，第二个指向第三个。

而不同层次之间通过父亲指针和儿子指针连接，父节点通过**子孙指针**指向下一层的首节点，而下一层的所有子节点通过**父亲指针**指向父节点。

#### 数据结构

```cpp
typedef struct item {
	char name[10];//文件名
	int type;//文件类型（文件/文件夹）
	int size;//文件大小
	int code;//文件的状态码
	int  datab;//文件内容开始指针
	int datae;//文件内容结束指针
	int father;//父文件夹指针
	int son;//子目录指针
	int brother;//同级指针
}item;
```

![image-20191208162809241](C:\Users\HP\AppData\Roaming\Typora\typora-user-images\image-20191208162809241.png)

#### 实现功能

| login  | 用户登录 |
| :----: | :------: |
| logout | 拥护注销 |
| Create | 新建文件 |
| Delete | 删除文件 |
|  Open  | 打开文件 |
| Close  | 关闭文件 |
|  Read  |  读文件  |
| Write  |  写文件  |
|  Dir   |  列目录  |



#### 函数原型

- **void userlogin()** 用户登录
- **void userlogout()** 用户注销

- **void cd(string filepath)**  进入文件夹
- **void createfile(string filename)** 新建文件/文件夹
- **void removefile(string filename)**移除文件/文件夹
- **void fileopen(string filename)** 文件打开
- **void fileclose(string filename) ** 文件关闭
- **void readf(string filename) ** 读文件内容
- **void writef(string data, string filename)** 写文件内容
- **void dir()** 列出文件目录
- **int search(string file)** 在该层目录中搜索文件
- **int find_writeposition() ** 寻找下一个写入位置
- **void init()** 初始化磁盘（提前建好一些文件）

#### 功能实现

虚拟文件系统通过两个二进制文件实现，**disk**与**data**，disk用于记录文件系统的层次结构，data文件用于记录文件内容。文件指针为数据在文件中的偏移量，通过移动读写的位置，可以将目标数据读出与写入。

用全局变量**userstate**标识用户当前状态，未登录的状态下无法进行进一步操作。

用全局变量**currentcd**记录目前所处的层次信息，该值为某一层首节点的偏移量。currentcd为负时意味着进入的空文件夹，而currentcd表示的为已进入文件夹信息的偏移量。

用全局变量**dataposition**指示data文件继续写入的位置，为文件读写提供服务，每次程序运行时初始化为data文件的末尾，并随着文件写入持续增加。

##### CD命令

<img src="C:\Users\HP\AppData\Roaming\Typora\typora-user-images\image-20191208200245506.png" alt="image-20191208200245506" style="zoom:67%;" />

##### removef命令

![image-20191208223252849](C:\Users\HP\AppData\Roaming\Typora\typora-user-images\image-20191208223252849.png)

##### createf 命令

![image-20191208210517981](C:\Users\HP\AppData\Roaming\Typora\typora-user-images\image-20191208210517981.png)

### 源代码

```c++
#define _CRT_SECURE_NO_WARNINGS
#include<iostream>
#include<fstream>
#include<stdlib.h>
#include<string>
#include<cstring>
#include<queue>
using namespace std;

fstream mydisk;
fstream mydata;
int userstate = 0;
int currentcd = 0;
int dataposition = 0;

typedef struct item {
	char name[10];//文件名
	int type;//文件类型（文件/文件夹）
	int size;//文件大小
	int code;//文件的状态码
	int  datab;//文件内容开始指针
	int datae;//文件内容结束指针
	int father;//父文件夹指针
	int son;//子目录指针
	int brother;//同级指针
}item;

void init() {

	item file1;//1
	strcpy(file1.name, "bin");
	file1.brother = -2;
	file1.son = 0;
	file1.father = -1;
	file1.datab = 0;
	file1.datae = 0;
	file1.size = 1;
	file1.code = 1;
	file1.type = 0;
	int position_file1 = mydisk.tellp();
	mydisk.write((char*)&file1, sizeof(file1));

	item file2;//1.1
	int position_file2 = mydisk.tellp();
	strcpy(file2.name, "bin.txt");
	file2.father = position_file1;
	file2.brother = -2;
	file2.son = -1 * position_file2;
	file2.datab = 0;
	file2.datae = 0;
	file2.size = 1;
	file2.code = 0;
	file2.type = 1;

	mydisk.write((char*)&file2, sizeof(file2));



	item file3;//1.2
	int position_file3 = mydisk.tellp();
	strcpy(file3.name, "bin2.txt");
	file3.father = position_file1;
	file3.brother = -2;
	file3.son = -1 * position_file3;
	file3.datab = 0;
	file3.datae = 0;
	file3.size = 1;
	file3.code = 1;
	file3.type = 1;

	mydisk.write((char*)&file3, sizeof(file3));

	item file4;//2
	int position_file4 = mydisk.tellp();
	strcpy(file4.name, "help.txt");
	file4.father = -1;
	file4.brother = -2;
	file4.son = -1 * position_file4;
	file4.datab = 0;
	file4.datae = 0;
	file4.size = 1;
	file4.code = 0;
	file4.type = 1;

	mydisk.write((char*)&file4, sizeof(file4));


	mydisk.seekp(position_file2);
	file2.brother = position_file3;
	mydisk.write((char*)&file2, sizeof(file2));

	mydisk.seekp(position_file1);
	file1.son = position_file2;
	file1.brother = position_file4;
	mydisk.write((char*)&file1, sizeof(file1));

	return;
}

int find_writeposition() {
	int p1, p2;
	item * tempfile = (item*)malloc(sizeof(item));
	mydisk.seekg(0, ios::beg);
	mydisk.read((char*)tempfile, sizeof(item));
	while (true)
	{
		p1 = mydisk.tellg();
		mydisk.read((char*)tempfile, sizeof(item));
		p2 = mydisk.tellg();
		if (-1 == p2)
		{
			free(tempfile);
			mydisk.clear();
			return p1;
		}
	}

	return mydisk.tellg();
}
void dir() {

	cout << ".." << endl;

	if (currentcd < 0) {
		if (currentcd == -1) {
			currentcd = -1*find_writeposition();
		}
		return;
	}
	mydisk.seekg(currentcd, ios::beg);
	mydisk.seekp(currentcd, ios::beg);

	printf("%-10s%-10s%-10s%-10s\n", "Name", "Size","Type","State");
	int a = mydisk.tellg();
	item *temp = (item*)malloc(sizeof(item));
	if (mydisk.read((char*)temp, sizeof(item)))

		printf("%-10s%-10d%-10d%-10d\n", temp->name, int(temp->size), int(temp->type), int(temp->code));
	while (int(temp->brother)!=-2)
	{
		mydisk.seekg(int(temp->brother), ios::beg);
		mydisk.read((char*)temp, sizeof(item));
		printf("%-10s%-10d%-10d%-10d\n", temp->name, int(temp->size), int(temp->type), int(temp->code));
	}
	free(temp);
	return;
}

int search(string file) {
	int posison;
	queue<item> filetree;

	mydisk.seekp(currentcd, ios::beg);
	mydisk.seekg(currentcd, ios::beg);

	item * tempfile = (item*)malloc(sizeof(item));

	mydisk.read((char*)tempfile, sizeof(item));

	filetree.push(*tempfile);

	mydisk.seekp(currentcd, ios::beg);
	mydisk.seekg(currentcd, ios::beg);

	int headposition = currentcd;
	while (!filetree.empty())
	{
		item filehead = filetree.front();

		if (filehead.name == file) {
			return headposition;

		}
		if (filehead.brother != 0) {
			headposition = filehead.brother;
			mydisk.seekg(filehead.brother, ios::beg);
			mydisk.read((char*)tempfile, sizeof(item));
			filetree.push(*tempfile);
		}

		filetree.pop();

	}
	return -1;
}

void fileclose(string filename) {
	int position = search(filename);
	if (position == -1) {
		cout << "Filename Error" << endl;
		return;
	}
	mydisk.seekg(position, ios::beg);
	item * targetfile = (item*)malloc(sizeof(item));

	mydisk.read((char*)targetfile, sizeof(item));
	targetfile->code = 0;

	mydisk.seekp(position, ios::beg);
	mydisk.write((char*)targetfile, sizeof(item));
	cout << "File Close!" << endl;

}

void fileopen(string filename) {
	int position = search(filename);
	if (position == -1) {
		cout << "Filename Error" << endl;
		return;
	}
	mydisk.seekg(position, ios::beg);
	item * targetfile = (item*)malloc(sizeof(item));

	mydisk.read((char*)targetfile, sizeof(item));
	targetfile->code = 1;

	mydisk.seekp(position, ios::beg);
	mydisk.write((char*)targetfile, sizeof(item));
	cout << "File Open!" << endl;

}







void writef(string data, string filename) {
	int position = search(filename);
	if (position == -1) {
		cout << "Not exist!" << endl;
		return;
	}
	item*tempfile = (item*)malloc(sizeof(item));
	mydisk.seekg(position, ios::beg);
	mydisk.read((char*)tempfile, sizeof(item));

	if (tempfile->type == 0) {
		cout << "Filetype Error!" << endl;
		return;
	}
	if (tempfile->code == 0) {
		cout << "Filestate Error!" << endl;
		return;
	}

	tempfile->datab = dataposition;
	tempfile->datae = dataposition + data.length();
	tempfile->size = data.length()+1;

	mydisk.seekp(position, ios::beg);
	mydisk.write((char *)tempfile, sizeof(item));

	position = tempfile->father;
	while (position!=-1)//***************
	{
		mydisk.seekg(position, ios::beg);
		mydisk.read((char*)tempfile, sizeof(item));

		tempfile->size+=data.length()+1;
		mydisk.seekp(position, ios::beg);
		mydisk.write((char *)tempfile, sizeof(item));
		position = tempfile->father;
	}

	mydata.seekp(dataposition, ios::beg);
	mydata.write(data.c_str(), data.length() + 1);
	dataposition += data.length() + 1;
	free(tempfile);
}



void readf(string filename) {

	int position = search(filename);
	if (position == -1) {
		cout << "Not exist!" << endl;
		return;
	}
	item*tempfile = (item*)malloc(sizeof(item));

	string data;
	mydisk.seekg(position, ios::beg);
	mydisk.read((char*)tempfile, sizeof(item));

	if (tempfile->type == 0) {
		cout << "Filetype Error!" << endl;
		return;
	}
	if (tempfile->code == 0) {
		cout << "Filestate Error!" << endl;
		return;
	}
	mydata.seekg(tempfile->datab, ios::beg);
	mydata.read((char*)data.c_str(), tempfile->datae - tempfile->datab + 1);
	printf("\033[2J");
	printf("%s\n", data.c_str());
	free(tempfile);
}


void createfile(string filename) {

	item * tempfile = new(item);
	item * tempfatherfile = new(item);
	strcpy(tempfile->name, filename.c_str());

	tempfile->father = -1;
	tempfile->brother = -2;
	tempfile->son = 0;
	tempfile->datab = 0;
	tempfile->datae = 0;
	tempfile->size = 1;
	tempfile->code = 0;


	if (filename.find(".") == string::npos)
		//文件类型判断
		tempfile->type = 0;
	else
		tempfile->type = 1;


	int lastposition = find_writeposition();//因为为新建文件 ，需要寻找写入位置


	if (currentcd < 0) {
		if (currentcd != -1) {
			mydisk.seekg(-1 * currentcd, ios::beg);
			mydisk.seekp(-1 * currentcd, ios::beg);
			mydisk.read((char*)tempfatherfile, sizeof(item));

			tempfile->father = -1 * currentcd;
			tempfile->brother = -2;
			tempfile->son = -1 * lastposition;

			mydisk.seekp(lastposition, ios::beg);
			mydisk.write((char*)tempfile, sizeof(item));

			tempfatherfile->son = lastposition;
			mydisk.seekp(-1 * currentcd, ios::beg);
			mydisk.write((char*)tempfatherfile, sizeof(item));
			currentcd = lastposition;
		}
		else {
			currentcd = lastposition;
			mydisk.seekg( currentcd, ios::beg);
			mydisk.seekp( currentcd, ios::beg);

			mydisk.write((char*)tempfile, sizeof(item));
			currentcd = lastposition;
		}
		
	}
	else if (currentcd == 0) {
		mydisk.seekg(currentcd, ios::beg);
		mydisk.seekp(currentcd, ios::beg);

		mydisk.read((char*)tempfatherfile, sizeof(item));

		tempfile->father = -1;
		tempfile->brother = tempfatherfile->brother;;
		tempfile->son = -1 * lastposition;

		mydisk.seekp(lastposition, ios::beg);
		mydisk.write((char*)tempfile, sizeof(item));

		tempfatherfile->brother = lastposition;
		mydisk.seekp(currentcd, ios::beg);
		mydisk.write((char*)tempfatherfile, sizeof(item));
		currentcd = 0;
	}
	else {
		mydisk.seekg(currentcd, ios::beg);
		mydisk.seekp(currentcd, ios::beg);

		mydisk.read((char*)tempfatherfile, sizeof(item));

		tempfile->father = tempfatherfile->father;
		tempfile->brother = tempfatherfile->brother;
		tempfile->son = -1 * lastposition;

		mydisk.seekp(lastposition, ios::beg);
		mydisk.write((char*)tempfile, sizeof(item));

		tempfatherfile->brother = lastposition;
		mydisk.seekp(currentcd, ios::beg);
		mydisk.write((char*)tempfatherfile, sizeof(item));
	}

	free(tempfile);
	free(tempfatherfile);

}


void removefile(string filename) {

	int position = search(filename);
	item * tempfile = new(item);
	item * tempfatherfile = new(item);

	mydisk.seekg(position, ios::beg);
	mydisk.read((char*)tempfile, sizeof(item));//当前节点

	if (position == currentcd) {//是否为第一层节点
		if (tempfile->father == -1) {			
		}
		else {
			mydisk.seekg(tempfile->father, ios::beg);
			mydisk.read((char*)tempfatherfile, sizeof(item));

			tempfatherfile->son = tempfile->brother;
			mydisk.seekp(tempfile->father, ios::beg);
			mydisk.write((char*)tempfatherfile, sizeof(item));
			
		}
		currentcd = tempfile->brother;
	}

	else {
		mydisk.seekg(currentcd, ios::beg);
		mydisk.read((char*)tempfatherfile, sizeof(item));

		while (tempfatherfile->brother != position)
		{
			mydisk.seekg(tempfatherfile->brother, ios::beg);
			mydisk.read((char*)tempfatherfile, sizeof(item));
		}

		tempfatherfile->brother = tempfile->brother;

		int p = search(tempfatherfile->name);

		mydisk.seekp(p, ios::beg);
		mydisk.write((char*)tempfatherfile, sizeof(item));
	}
	free(tempfile);
	free(tempfatherfile);
}

void cd(string filepath) {

	item * tempfile = (item*)malloc(sizeof(item));
	if (filepath != "..") {
		if (currentcd < 0)
		{
			if (currentcd == -1) currentcd = find_writeposition();
			else currentcd *= -1;
		}
		int position = search(filepath);

		mydisk.seekg(position, ios::beg);

		mydisk.read((char*)tempfile, sizeof(item));

		if (tempfile->type == 0) {
			currentcd = tempfile->son;
		}
		else
		{
			cout << "Filepath Error!" << endl;
		}
	}
	else {
		if (currentcd < 0)
		{
			mydisk.seekg(-1 * currentcd, ios::beg);
			mydisk.read((char*)tempfile, sizeof(item));

			currentcd = tempfile->father;

			if (currentcd != -1) {//非第一层
				mydisk.seekg(currentcd, ios::beg);
				mydisk.read((char*)tempfile, sizeof(item));
				currentcd = tempfile->son;
			}
			else
			{
				currentcd = 0;
			}
		}
		else if (currentcd > 0) {
			mydisk.seekg(currentcd, ios::beg);
			mydisk.read((char*)tempfile, sizeof(item));

			currentcd = tempfile->father;
			mydisk.seekg(currentcd, ios::beg);
			mydisk.read((char*)tempfile, sizeof(item));

			currentcd = tempfile->father;

			if (currentcd != -1) {
				mydisk.seekg(currentcd, ios::beg);
				mydisk.read((char*)tempfile, sizeof(item));
				currentcd = tempfile->son;
			}
			else {
				currentcd = 0;
			}
		}


	}
	return;
}

void userlogin() {
	fstream user;
	user.open("./user", ios::binary | ios::in | ios::out);

	string inputname;
	cout << "login as: ";
	cin >> inputname;
	string inputpassword;
	cout << "password: ";
	cin >> inputpassword;
	string userdata;
	getline(user, userdata);
	int index1 = userdata.find("[");
	int index2 = userdata.find("]");
	string username(userdata.begin() + index1 + 1, userdata.begin() + index2);
	int index3 = userdata.find("[", index2);
	string password(userdata.begin() + index3 + 1, userdata.end() - 1);
	if (inputname == username && inputpassword == password) {
		cout << "Welcome " << endl;
		userstate = 1;
	}
	else {
		cout << "Access Denied" << endl;
	}
	user.close();
}

void userlogout() {
	userstate = 0;
	cout << "GoodBye\n";
}

int main() {

	mydisk.open("./disk", ios::binary | ios::in | ios::out);
	//init();


	while (1)
	{
		cout << "> ";
		string cmd;
		cin >> cmd;
		if (cmd == "dir") {
			if (userstate == 0) {
				printf("Userstate Error!\n");
			}
			else {
				dir();
				cout << endl;
			}
		}
		else if (cmd == "login") {
			userlogin();
		}
		else if (cmd == "logout") {
			userlogout();
		}
		else if (cmd == "closef") {
			if (userstate == 0) {
				printf("Userstate Error!\n");
			}
			else {
				string filename;
				cout << "Input Filename:";
				cin >> filename;
				fileclose(filename);

			}
		}
		else if (cmd == "openf") {
			if (userstate == 0) {
				printf("Userstate Error!\n");
			}
			else {
				string filename;
				cout << "Input Filename:";
				cin >> filename;
				fileopen(filename);

			}
		}
		else if (cmd == "mkdir") {
			if (userstate == 0) {
				printf("Userstate Error!\n");
			}
			else {
				string filename;
				cout << "Input Filename:";
				cin >> filename;
				createfile(filename);

			}
		}
		else if (cmd == "rmdir") {
			if (userstate == 0) {
				printf("Userstate Error!\n");
			}
			else {
				string filename;
				cout << "Input Filename:";
				cin >> filename;
				removefile(filename);

			}
		}
		else if (cmd == "writef") {
			if (userstate == 0) {
				printf("Userstate Error!\n");
			}
			else {

				mydata.open("./data", ios::binary | ios::app | ios::out);
				mydata.seekp(0, ios::end);
				dataposition = int(mydata.tellp());
				string data, filename;
				cout << "Filename:";
				cin >> filename;
				cout << "Filedata:";
				cin >> data;
				writef(data, filename);
				mydata.close();

			}


		}
		else if (cmd == "readf") {
			if (userstate == 0) {
				printf("Userstate Error!\n");
			}
			else {
				mydata.open("./data", ios::binary | ios::in);
				mydata.seekg(0, ios::end);
				dataposition = int(mydata.tellg());
				string filename;
				cout << "Filename:";
				cin >> filename;
				readf(filename);
				mydata.close();
			}

		}
		else if (cmd == "exist") break;
		else if (cmd == "cd") {
			if (userstate == 0) {
				printf("Userstate Error!\n");
			}
			else {
				string filename;
				cout << "Input Filename:";
				cin >> filename;
				cd(filename);

			}
		}

	}

	mydisk.close();
	return 0;
}
```







