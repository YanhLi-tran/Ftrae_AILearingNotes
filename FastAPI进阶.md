# **中间件**

使用中间件可以为每个请求前后添加统一的处理逻辑，使用场景：

- 身份认证
- 日志记录
- 跨域处理
- 响应头处理
- 性能监控

中间件 (Middleware) 是一个在每次请求进入FastAPI应用是都会被执行的函数。

它在请求到达实际的路径操作 (路由处理函数) 之前运行，并且在响应返回给客户端之前再运行一次。

![image-20260625151624733](images/image-20260625151624733.png)

中间件A和中间件B的执行顺序：自下而上进行执行

## **代码示例**

函数的顶部使用装饰器@app.middleware("http")

```python
@app.middleware("http")
# request是请求，call_next的作用是传递请求给路径处理函数
async def middleware1(request, call_next):
    print('中间件1 -- start')
    # await是异步处理响应等待
    response = await call_next(request)
    print('中间件1 -- end')
    return response

@app.middleware("http")
async def middleware2(request, call_next):
    print('中间件2 -- start')
    # await是异步处理响应等待
    response = await call_next(request)
    print('中间件2 -- end')
    return response
```

![image-20260625151744739](images/image-20260625151744739.png)

会先运行中间件2, 然后才运行中间件1, 自下而上的运行顺序。

# **依赖注入**

使用依赖注入系统来共享通用逻辑，减少代码重复

- 依赖项：可重用的组件 (函数/类) ，负责提供某种功能或数据。
- 注入：FastAPI自动帮你调用依赖项，并将结果“注入”到路径操作函数中

优点：

- 代码复用：一次编写，多处使用
- 解耦：业务逻辑与基础设施代码分离
- 易于测试：轻松地用模拟依赖替换真实依赖进行测试

依赖注入应用场景：

1. 处理请求参数：从请求中提取和验证参数 (路径参数、查询参数、请求体)
2. 共享业务逻辑：抽取封装多个路由公用的逻辑代码
3. 共享数据库连接：管理数据库会话的创建、使用、关闭
4. 安全和认证：验证用户身份、检查权限和角色要求等

依赖注入步骤：

1. 创建依赖项

2. 导入Depends

   ```python
   from fastapi import FastAPI,Depends
   ```

3. 声明依赖项

## **代码示例**

```python
# 分页参数逻辑公用：新闻列表和用户列表

# ding'yi'yi
async def common_parameters(
	skip: int = Query(0,ge = 0)
    limit: int = Query(10,le = 60)
):
    return {"skip":skip,"limit":limit}

# 注入需要的接口
@app.get("/news/news_list")
async def get_news_list(commons = Depends(common_parameters)):
    return commons

@app.get("/user/user_list")
async def get_user_list(commons = Depends(common_parameters)):
    return commons
```

# **ORM简介**

ORM (Object-RelationalMapping，对象关系映射) 是一种编程技术，用于在面向对象编程语言和关系型数据库之间建立映射。它允许开发者通过操作对象的方式与数据库进行交互，而无需直接编写复杂的SQL语句。

ORM优势：

- 减少重复的SQL代码
- 代码更简洁易读
- 自动处理数据库连接和事务
- 自动防止SQL注入攻击

ORM分类：

| 排名 | ORM工具              | 特点                     | 适应场景                  |
| ---- | -------------------- | ------------------------ | ------------------------- |
| 1    | SQLAlchemy ORM ( √ ) | 功能最强、最灵活、企业级 | 各类API、微服务、数据应用 |
| 2    | Django ORM           | 封装好、上手快           | Django项目、管理后台      |
| 3    | Tortoise             | 全异步                   | 异步Web服务、高并发API    |

安装SQLAlchemy命令：

```shell
pip install sqlalchemy[asyncio] aiomysql
```

## **建库**

```python
# 库名xx
create database xx;
```

## **建表**

### **步骤**

1. 创建数据库引擎
2. 定义模型类
3. 启动应用时建表

#### **创建数据库引擎**

需要导入create_async_engine

```python
from sqlalchemy.ext.asyncio import create_async_engine
```

##### **创建异步引擎**

```python
# 数据库+驱动://用户名:密码@localhost:3306/数据库名?charset=utf8
ASYNC_DATABASE_URL = "mysql+aiomysql://root:123456@localhost:3306/fastapi_test?charset=utf8"

async_engine = create_async_engine(
	ASYNC_DATABASE_URL,
    echo=True,			#可选：输出SQL日志
    pool_size = 10,		#设置连接池中保持的持久连接数
    max_overflow = 20	#设置连接池允许创建的额外连接数
)
```

#### **定义模型类**

- 基类，继承DeclarativeBase (包含通用属性和字段的映射)
- 定义数据库表对应的模型类

需要导入DeclarativeBase,Mapped,mapped_column

```python
from sqlalchemy.orm import DeclarativeBase,Mapped,mapped_column
```

##### **基类代码**

```python
from sqlalchemy import func, DateTime
from datetime import datetime

# 基类：创建时间，更新时间
class Base(DeclarativeBase):
    create_time:Mapped[datetime] = mappen_column(
    	DateTime,
        insert_default=func.now(),
        default=func.now,
        comment="创建时间"
    )
    update_time:Mapped[datetime] = mapped_column(
        Datatime,
        insert_default=func.now(),
        onupdate=func.now(),
        default=func.now,
        comment="修改时间"
    )
```

##### **模型类代码**

```python
# 书籍表：id、书名、作者、价格、出版社
class Book(Base):
    __tablename__="book"
    
    id: Mapped[int] = mapped_column(primary_key=True,comment="书籍id")
    bookname: Mapped[str] = mapped_column(String(255),comment="书名")
    author: Mapped[str] = mapped_column(String(255),comment="作者")
    price: Mapped[float] = mapped_column(Float,comment="价格")
    publisher: Mappen[str] = mapped_column(String(255),comment="出版社")
```

#### **启动应用时建表**

- 从连接池获取异步连接，开启事务，执行ORM操作
- FastAPI应用启动时，创建数据库表

##### **代码示例**：

```python
async def create_tables():
	# 获取异步引擎，创建事务
    async with async_engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)	# 使用Base模型类的元数据创建

@app.on_event("startup")
async def startup_event():
    await create_tables()
```

### **练习**

![image-20260624183830747](images/image-20260624183830747.png)

```python
from sqlalchemy.orm import DeclarativeBase,Mapped,mapped_column
from sqlalchemy.ext.asyncio import create_async_engine
from sqlalchemy import func, DateTime,String
from contextlib import asynccontextmanager
from datetime import datetime
from fastapi import FastAPI

ASYNC_DATABASE_URL = "mysql+aiomysql://root:123456@localhost:3306/user?charset=utf8"

# 创建异步引擎
async_engine=create_async_engine(ASYNC_DATABASE_URL,echo=True,pool_size=10,max_overflow=20)

# 创建基类
class Base(DeclarativeBase):
    create_time: Mapped[datetime] = mapped_column(
        DateTime,
        default=func.now(),
        comment="创建时间"
    )
    update_time: Mapped[datetime] = mapped_column(
        DateTime,
        onupdate=func.now(),
        default=func.now(),
        comment="更新时间"
    )

# 创建模型类
class User(Base):
    __tablename__ = "user"
    id: Mapped[int] = mapped_column(primary_key=True,comment="用户id")
    username: Mapped[str] = mapped_column(String(8),unique=True,comment="用户名")
    password: Mapped[str] = mapped_column(String(20),comment="用户密码")

# 使用 lifespan 管理应用生命周期(替代已弃用的on_event)
@asynccontextmanager
async def lifespan(app: FastAPI):
    # 启动时创建数据库表（若表已存在则忽略）
    print("应用启动，创建数据库表")
    async with async_engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield

    # 关闭时释放数据库连接池资源
    print("应用关闭，清理资源......")
    await async_engine.dispose()

app = FastAPI(lifespan=lifespan)
```

## **路由匹配当中使用ORM**

核心：创建依赖项获取数据库会话，使用Depends注入到路由处理函数

需要导入数据库会话工厂函数async_sessionmaker和会话类AsyncSession：

```python
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker,AsyncSession
```

***需求：查询功能的接口，查询图书***

### **创建依赖项**

#### **代码示例**

```python
# 创建异步会话工厂
AsyncSessionLocal = async_sessionmaker(
	bind = async_engine,		# 绑定数据库引擎 
    class= AsyncSession,		# 指定会话类
    expire_on_commit = False	#提交后会话不过期，不会重新查询数据库
)

# 创建依赖项
async def get_database():
    async with AsyncSessionLocal() as session:
        try:
            yield session				# 返回数据库会话给路由处理函数
            await session.commit()		# 提交事务
        except Exception:
            await session.rollback()	# 有异常，回滚
            raise
        finally:
            await session.close()		# 关闭会话
```

### **注入路由处理函数**

#### **代码示例**

```python
# 创建路由
@app.get("/book/books")
async def get_book_list(db: AsyncSession = Depends(get_database)):
    # 查询是否注入成功
    result = await db.excute(select(Book))
    book = result.scalars().all()
    return book
```

## **数据库操作**

- 查询 select()
- 新增 add()
- 更新 (先查再改，重新赋值)
- 删除 delete()

### **查询**

核心语句：await db.execute(select(模型类 ))，返回一个ORM对象。

获取所有数据：

```python
scalars().all()
```

获取单条数据：

```python
scalars().first()
get(模型类,主键值)
```

#### **代码示例**

```python
@ app.get("book/books")
async def get_book_list(db: AsyncSession = Depends(get_database)):
	result = await db.execute(select(Book))	# 查询，返回一个ORM对象
    book = result.scalars().all()			# 获取所有图书信息
    book = result.scalars().first()			# 获取第一本图书信息
    await db.get(Book,5)					# 返回id为5的图书信息(根据主键进行查询)
    return book
```

#### 查询条件

核心语句：select(Book).where(条件1,条件2,...)

- 条件查询：比较判断有 == ; >= ; <= ; > ; <等
- 模糊查询：like()
- 与非查询：& ; | ; ~
- 包含查询：in_()
- 聚合查询：func.方法(模型类.属性)

##### **条件查询**

###### **代码示例**

```python
# 需求：路径参数是书籍id
@app.get("/book/get_book/{book_id}")
async def get_book_list(book_id: int,db: AsyncSession = Depends(get_database)):
    result = await db.execute(select(Book).where(Book.id=book_id))
    book = result.scalar_one_or_none()
    return book
```

```python
# 需求：价格大于等于200
@app.get("/book/search_book")
async def get_search_book(db: AsyncSession = Depends(get_database)):
    result = await db.execute(select(Book).where(Book.price >= 200))
    books = result.scalars().all()
    return books
```

##### **模糊查询**

- % ：零个、一个或多个字符
- _ ：一个单个字符

###### 代码示例

```python
# 需求：作者以“曹”开头
@app.get("/book/search_book")
async def get_search_book(db: AsyncSession = Depends(get_database)):
    result = await db.execute(select(Book).where(Book.author.like("曹%"))) # 模糊查询
    book = result.scalars().all()
    return book
```

##### **与非查询**

###### **代码示例**

```python
# 需求：作者以“曹”开头，且书籍价格大于100
@app.get("/book/search_book")
async def get_search_book(db: AsyncSession = Depends(get_database)):
    result = await db.execute(select(Book).where(Book.author.like("曹%") & (Book.price > 100)))	# 与非判断
    book = result.scalars().all()
    return book
```

##### 包含查询

###### **代码示例**

```python
# 需求：书籍id列表，数据库里面的id如果在书籍id列表里面就返回
id_list = [1,3,5,7]
@app.get("/book/search_book")
async def get_search_book(db: AsyncSession = Depends(get_database)):
    result = await db.execute(select(Book).where(Book.id.in_(id_list)))	# 包含查询
    book = result.scalars().all()
    return book
```

##### **聚合查询**

- count：统计行数量
- avg：求平均值
- max：求最大值
- min：求最小值
- sum：求和

###### **代码示例**

```python
@app.get("/book/count")
async def get_count(db: AsyncSession = Depends(get_database)):
    result = await db.execute(select(func.count(Book.id)))	# 求一共有多少本书
    result = await db.execute(select(func.sum(Book.price)))	# 求所有书的总价
    number = result.scalar()	# scalar()用来提取一个数值(标量值)
    return number
```

##### **分页查询**

核心语句：select().offset().limit()

- offset：跳过的记录数
  offset值 =  (当前页码-1)×每页数量limit
- limit：返回的记录数

###### **代码示例**

```python
# 
@app.get("/book/get_book_list")
async def get_book_list(
    page: int = 1,
    page_size: int = 3,
    db: AsyncSession = Depends(get_database)
):
    # offset = (页码-1)×每页数量
    skip = (page-1)*page_size
    # offset：跳过的记录数  limit：每页的记录数
    stmt = select(Book).offset(skip).limit(page_size)
    result = await db.execute(stmt)
    books = result.scalars().all()
    return books
```

### **增加**

核心步骤：定义ORM对象 -> 添加对象到事务：add(对象) -> commit提交到数据库

#### **代码示例**

```python
# 需求：用户输入图书信息(id、书名、作者、价格、出版社) -> 新增
# 用户输入 -> 参数 -> 请求体
# 使用的模型类是Book类
@app.post("/book/add_book")
async def add_book(book: Book，db: AsyncSession = Depends(get_database)):
    # ORM对象 -> add -> commit
    book_obj = book(**book.__dict__)
    db.add(book_obj)
    await db.commit
    return book
```

### **更新**

核心步骤：查询get -> 属性重新赋值 -> commit提交到数据库

#### **代码示例**

```python
# 需求：修改图书信息 先查再改
# 设计思路：路径参数书籍id：作用是查找
# 请求体参数：作用是新数据(书名、作者价格、出版社)
@app.put("book/update_book/{book_id}")
async def update_book(book_id: int,data: Book,db: AsyncSession = Depends(get_database)):
    # 查找图书
    db_book = await db.get(Book,book_id)
    
    # 如果未找到则抛出异常
    if db_book is None:
        raise HTTPException(
        	status_code=404,
            detail="查无此书"
        )
     
    # 找到了则修改，重新赋值
    db_book.name = data.name
    db_book.writer = data.writer
    db_book.price = data.price
    db_book.publisher = data.publisher
    
    # 提交到数据库
    await db.commit()
    return db_book
```

### **删除**

核心步骤：查询 get -> delete 删除 -> commit 提交到数据库

#### **代码示例**

```python
# 
@app.delete("/book/delete_book/{book_id}")
async def delete_book(book_id: int,db: AsyncSession = Depends(get_database)):
    # 先查再删最后提交
    db_book = await db.get(Book,book_id)
    
    if db_book is None:
        raise HTTPException(
        	status_code=404,
            detail="查无此书"
        )
        
     await db.delete(db_book)
     await db.commit()
     return {"msg":"删除图书成功"}
```

