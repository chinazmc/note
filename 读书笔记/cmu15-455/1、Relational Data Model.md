
## 

为什么需要 Database？[](https://zhenghe.gitbook.io/open-courses/cmu-15-445-645-database-systems/relational-data-model#wei-shen-me-xu-yao-database)

假设我们需要存储：

-   艺术家（Artists）信息
    

-   艺术家发行的专辑（Albums）信息
    

我们可以用两个 CSV 文件来分别存储艺术家和专辑信息，然后用程序来解析和序列化相关数据：

艺术家信息表：

Artist.csv

"Wu Tang Clan",1992,"USA"

"Notorious BIG",1992,"USA"

"Ice Cube",1989,"USA"

专辑信息表：

Album.csv

"Enter the Wu Tang","Wu Tang Clan",1993

"St.Ides Mix Tape","Wu Tang Clan",1994

"AmeriKKKa's Most Wanted","Ice Cube",1990

假设我们想知道 “Ice Cube 在哪年首发”，就会写这样的查询脚本：

for line in file:

record = parse(line)

if record[0] == "Ice Cube":

print(int(record[1]))

但这种简单的方案有很多缺陷：

-   数据的质量方面
    
    -   很难保证同一个艺术家发行的每条专辑信息中，艺术家字段一致
        
    
    -   很难阻止用户在年份字段写入不合法的字符串
        
    
    -   很难优雅地处理一张专辑由多个艺术家共同发行的情况
        
    

-   实现方面
    
    -   查询一条特定的记录，效率低
        
    
    -   当有多个应用使用该 CSV 数据库
        
        -   查询脚本需要重复写
            
        
        -   多个线程一起写时，如何保证数据一致性
            
        
    

-   数据持久
    
    -   正在写记录时遇到宕机
        
    
    -   如何复制数据库到多台机器上来保证高可用性
        
    

以上缺陷迫使我们需要升级 CSV 数据库，于是就有了专业的数据库系统（DBMS）。

## 

DBMS 的提出[](https://zhenghe.gitbook.io/open-courses/cmu-15-445-645-database-systems/relational-data-model#dbms-de-ti-chu)

### 

分离逻辑层和物理层[](https://zhenghe.gitbook.io/open-courses/cmu-15-445-645-database-systems/relational-data-model#fen-li-luo-ji-ceng-he-wu-li-ceng)

所有系统都会产生数据，因此数据库几乎是所有系统都不可或缺的模块。在早期，各个项目各自造轮子，因为每个轮子都是为应用量身打造，这些系统的逻辑层（logical）和物理层（physical）普遍耦合度很高。

Ted Codd 发现这个问题后，提出 DBMS 的抽象（Abstraction）：

-   用简单的、统一的数据结构存储数据
    

-   通过高级语言操作数据
    

-   逻辑层和物理层分离，系统开发者只关心逻辑层，而 DBMS 开发者才关心物理层。
    

### 

数据模型[](https://zhenghe.gitbook.io/open-courses/cmu-15-445-645-database-systems/relational-data-model#shu-ju-mo-xing)

在逻辑层中，我们通常需要对所需存储的数据进行建模。如今，市面上有的数据模型包括：

-   Relational => 大部分 DBMS 属于关系型，也是本课讨论的重点
    

-   Key/Value
    

-   Graph
    

-   Document
    

-   Column-family
    

-   Array/Matrix
    

### 

Relational Model[](https://zhenghe.gitbook.io/open-courses/cmu-15-445-645-database-systems/relational-data-model#relational-model)

#### 

Relation & Tuple[](https://zhenghe.gitbook.io/open-courses/cmu-15-445-645-database-systems/relational-data-model#relation-and-tuple)

每个 Relation 都是一个无序集合（unordered set），集合中的元素称为 tuple，每个 tuple 由一组属性构成，这些属性在逻辑上通常有内在联系。

#### 

Primary Keys[](https://zhenghe.gitbook.io/open-courses/cmu-15-445-645-database-systems/relational-data-model#primary-keys)

primary key 在一个 Relation 中唯一确定一个 tuple，如果你不指定，有些 DBMSs 会自动帮你生成 primary key。

#### 

Foreign Keys[](https://zhenghe.gitbook.io/open-courses/cmu-15-445-645-database-systems/relational-data-model#foreign-keys)

foreign key 唯一确定另一个 relation 中的一个 tuple

利用这些基本概念，我们就可以利用第三张表，ArtistAlbum，来解决专辑与艺术家的 1 对多的关系问题：

Artist

id, name, year, country

Album

id, name, year

ArtistAlbum

artist_id, album_id

#### 

Data Manipulation Languages (DML)[](https://zhenghe.gitbook.io/open-courses/cmu-15-445-645-database-systems/relational-data-model#data-manipulation-languages-dml)

在 Relational Model 中从数据库中查询数据通常有两种方式：Procedural 与 NonProcedural：

-   Procedural：查询命令需要指定 DBMS 执行时的具体查询策略，如 Relational Algebra
    

-   Non-Procedural：查询命令只需要指定想要查询哪些数据，无需关心幕后的故事，如 SQL
    

使用哪种方式是具体的实现问题，与 Relational Model 本身无关。

#### 

Relational Algebra[](https://zhenghe.gitbook.io/open-courses/cmu-15-445-645-database-systems/relational-data-model#relational-algebra)

relational algebra 是基于 set algebra 提出的，从 relation 中查询和修改 tuples 的一些基本操作，它们包括：
![[Pasted image 20230307212621.png]]
    

将这些操作串联起来，我们就能构建更复杂的操作

**注意**：使用 Relation Algebra 时，我们实际上指定了执行策略，如：
![[Pasted image 20230307212645.png]]
它们所做的事情都是 ”返回 R 和 S Join 后的结果中，b_id 等于 102 的 tuples“。

虽然 Relational Algebra 只是 Relational Model 的具体实现方式，但在之后的课程将会看到它对查询优化、执行的帮助。

## 

参考[](https://zhenghe.gitbook.io/open-courses/cmu-15-445-645-database-systems/relational-data-model#can-kao)

​[slides](https://15445.courses.cs.cmu.edu/fall2018/slides/01-introduction.pdf)​

​

​

**​**