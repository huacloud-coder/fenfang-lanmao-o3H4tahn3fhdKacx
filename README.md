
我们在定义SQLAlchemy对象模型的关系的时候，用到了relationship 来标识关系，其中 lazy 的参数有多种不同的加载策略，本篇随笔介绍它们之间的关系，以及在异步处理中的一些代码案例。


### 1、在 SQLAlchemy 中定义关系


在 SQLAlchemy 中，`relationship()` 函数用于定义表之间的关系（如 `one-to-many`、`many-to-one`、`many-to-many` 等）。它支持许多参数来控制如何加载和处理关联的数据。以下是一些常用的 `relationship()` 参数及其说明：


#### 1\. **`lazy`**


* **作用**: 控制如何加载关联数据。
* **可选值**:
	+ `'select'`: 延迟加载。访问关系属性时，发送一个独立的查询来获取关联数据（默认值）。
	+ `'selectin'`: 使用 `IN` 查询批量加载关联对象，避免 n\+1 查询问题。
	+ `'joined'`: 使用 `JOIN` 直接在主查询中加载关联数据。
	+ `'subquery'`: 使用子查询来批量加载关联对象。
	+ `'immediate'`: 在加载主对象后，立即加载关联对象。
	+ `'dynamic'`: 仅适用于 `one-to-many`，返回一个查询对象，可以进一步过滤或操作关联数据。
* #### 详细说明


　　在SQLAlchemy中，`lazy`是一个定义ORM关系如何加载的参数，主要用于控制关联关系（如`one-to-many`、`many-to-one`等）在访问时的加载方式。


1）`lazy='select'` (默认)


* **说明**: 这是最常见的方式，使用"延迟加载"策略。当访问关联属性时，SQLAlchemy会发送一条新的SQL查询来加载相关数据。
* **优点**: 避免了不必要的查询，节省资源。
* **缺点**: 当你访问多个关联对象时，可能会导致"n\+1查询问题"，即每次访问关联数据时都会发出新的SQL查询。


2） `lazy='selectin'`


* **说明**: 类似于`lazy='select'`，但通过`IN`语句批量查询相关对象。SQLAlchemy会在一次查询中批量获取多个对象的关联数据，而不是为每个对象单独查询。
* **优点**: 解决了"n\+1查询问题"，效率高于`select`。
* **缺点**: 适用于可以通过`IN`语句高效查询的场景，但如果结果集非常大，可能会影响性能。


3） `lazy='joined'`


* **说明**: 在主查询时，使用`JOIN`语句直接加载关联对象。这意味着关联对象在查询时就会被立即加载，不需要额外的查询。
* **优点**: 避免了多个SQL查询，适合在同一查询中需要大量关联数据的场景。
* **缺点**: 如果`JOIN`的表数据较多，可能会导致查询结果变得复杂且性能下降。


4）`lazy='immediate'`


* **说明**: 在加载主对象时，立即加载所有关联对象。与`select`类似，但是在主对象加载后，马上发送查询请求加载关联对象。
* **优点**: 保证在对象加载后立刻有完整的数据。
* **缺点**: 对每个关联的对象仍然会发送单独的查询，可能造成"n\+1查询问题"。


5）`lazy='subquery'`


* **说明**: 使用子查询来加载关联对象。SQLAlchemy会在查询主对象时生成一个子查询，以批量加载相关对象。
* **优点**: 避免了"n\+1查询问题"，适合处理大型数据集。
* **缺点**: 子查询可能会导致查询效率降低，特别是在复杂的查询场景中。


6）`lazy='dynamic'`


* **说明**: 仅适用于`one-to-many`关系，返回一个查询对象，而不是实际的结果集。你可以通过调用查询对象来进一步过滤或操作关联对象。
* **优点**: 非常灵活，可以根据需要随时查询关联对象。
* **缺点**: 不能使用通常的方式访问关联属性，必须通过查询进一步获取数据。


 


#### 2\. **`backref`**


* **作用**: 定义反向引用，允许从关联表访问当前表。
* **用法**: 通过 `backref`，可以在关联的表中自动生成一个反向关系，避免手动定义双向关系。
* **示例**:




```
class Parent(Base):
    __tablename__ = 'parent'
    id = Column(Integer, primary_key=True)
    children = relationship("Child", backref="parent")

class Child(Base):
    __tablename__ = 'child'
    id = Column(Integer, primary_key=True)
    parent_id = Column(Integer, ForeignKey('parent.id'))
```


#### 3\. **`back_populates`**


* **作用**: 手动定义双向关系时，使用 `back_populates` 来明确地表示两个表之间的相互关系。
* **示例**:




```
class Parent(Base):
    __tablename__ = 'parent'
    id = Column(Integer, primary_key=True)
    children = relationship("Child", back_populates="parent")

class Child(Base):
    __tablename__ = 'child'
    id = Column(Integer, primary_key=True)
    parent_id = Column(Integer, ForeignKey('parent.id'))
    parent = relationship("Parent", back_populates="children")
```


#### 4\. **`cascade`**


* **作用**: 定义级联操作，决定在父对象上进行操作时，是否自动对关联的子对象进行相应操作。
* **常见值**:


	+ `'save-update'`: 当父对象被保存或更新时，子对象也会被保存或更新。
	+ `'delete'`: 当父对象被删除时，子对象也会被删除。
	+ `'delete-orphan'`: 当子对象失去与父对象的关联时，子对象将被删除。
	+ `'all'`: 包含所有级联操作。
* **示例**:




```
children = relationship("Child", cascade="all, delete-orphan")
```


#### 5\. **`uselist`**


* **作用**: 控制关联属性是否返回一个列表。适用于 `one-to-one` 和 `one-to-many` 关系。
* **用法**:


	+ `True`: 返回一个列表（适用于 `one-to-many`，默认值）。
	+ `False`: 返回单个对象（适用于 `one-to-one`）。
* **示例**:




```
parent = relationship("Parent", uselist=False)  # one-to-one 关系
```


#### 6\. **`order_by`**


* **作用**: 定义关联对象的排序方式。
* **示例**:




```
children = relationship("Child", order_by="Child.name")
```


#### 7\. **`foreign_keys`**


* **作用**: 显式指定哪些列是用于定义关联关系的外键，适用于存在多个外键的场景。
* **示例**:




```
parent = relationship("Parent", foreign_keys="[Child.parent_id]")
```


#### 8\. **`primaryjoin`**


* **作用**: 明确定义关联关系的连接条件，通常在 SQLAlchemy 无法自动推断时使用。
* **示例**:




```
parent = relationship("Parent", primaryjoin="Parent.id == Child.parent_id")
```


#### 9\. **`secondary`**


* **作用**: 定义多对多（`many-to-many`）关系时，指定关联的中间表。
* **示例**:




```
class Association(Base):
    __tablename__ = 'association'
    parent_id = Column(Integer, ForeignKey('parent.id'))
    child_id = Column(Integer, ForeignKey('child.id'))

children = relationship("Child", secondary="association")
```


#### 10\. **`secondaryjoin`**


* **作用**: 定义 `secondary` 表中的关联条件，通常用于复杂的多对多关系。
* **示例**:




```
children = relationship("Child", secondary="association", 
                        secondaryjoin="Child.id == Association.child_id")
```


#### 11\. **`viewonly`**


* **作用**: 定义只读的关系，不允许通过此关系修改数据。
* **示例**:




```
children = relationship("Child", viewonly=True)
```


#### 12\. **`passive_deletes`**


* **作用**: 控制删除时的行为。如果设置为 `True`，SQLAlchemy 不会主动删除关联对象，而是依赖数据库的级联删除。
* **示例**:




```
children = relationship("Child", passive_deletes=True)
```


这些参数可以根据具体的业务需求和场景进行调整，以优化查询和数据管理策略。


 


### 2、用户角色表的关系分析


在实际业务中，机构和用户是多对多的关系的，我们以机构表定义来进行分析它们的关系信息。


如机构表的模型定义大致如下。




```
class Ou(Base):
    """机构（部门）信息-表模型"""
    __tablename__ = "t_acl_ou"
    id = Column(Integer, primary_key=True, comment="主键", autoincrement=True)
    pid = Column(Integer, ForeignKey("t_acl_ou.id"), comment="父级机构ID", default="-1")
    handno = Column(String, comment="机构编码")
    name = Column(String, comment="机构名称")

    # 定义 parent 关系
    parent = relationship(
        "Ou", remote_side=[id], back_populates="children", lazy="immediate"
    )
    # 定义 children 关系
    children = relationship("Ou", back_populates="parent", lazy="immediate")
    # 定义 users 关系
    users = relationship(
        "User", secondary="t_acl_ou_user", back_populates="ous", lazy="select"
    )
```


我们可以看到其中加载的多对多关系是采用lazy\=select的方式的。


当你使用 `await session.get(Ou, ou_id)` 来获取一个 `Ou` 对象后，访问其关系属性（如 `ou.users`）时，可能会遇到异步相关的问题。原因是，SQLAlchemy 的异步会话需要使用 `selectinload` 或其他异步加载选项来确保在异步环境中正确地加载关联数据。


在默认的 `lazy='select'` 关系中，加载关系对象会触发一个同步查询，而这与异步会话不兼容，导致错误。为了解决这个问题，你需要确保关系的加载是通过异步的方式进行的。


#### 解决方法：


**1\. 使用 `selectinload` 进行预加载**


在查询时，显式地通过 `selectinload` 来加载关联的 `users` 关系：




```
from sqlalchemy.orm import selectinload

ou = await session.get(Ou, ou_id, options=[selectinload(Ou.users)])

# 现在你可以访问 ou.users，关系对象已经被异步加载
print(ou.users)
```


 


**2\. 使用 `lazy='selectin'` 或其他异步兼容的加载策略**


你还可以在定义模型的关联关系时，将 `lazy='selectin'` 设置为默认的加载方式，这样当访问关联属性时，SQLAlchemy 会自动使用异步兼容的加载机制：




```
class Ou(Base):
    __tablename__ = 'ou'
    id = Column(Integer, primary_key=True)
    users = relationship("User", lazy='selectin')  # 使用 selectin 异步加载

ou = await session.get(Ou, ou_id)
print(ou.users)  # 关联对象可以正常异步访问
```


#### 总结：


* 在异步环境中访问关系对象时，如果使用了同步的 `lazy='select'`，会导致异步不兼容问题。
* 解决方案是通过查询时使用 `selectinload` 或将关系的 `lazy` 属性设置为异步兼容的选项，如 `selectin`。


因此，如果机构和用户的关系信息，我们可以通过selectload关系实现加载，也可以考虑使用中间表的关系进行获取，如下代码所示：获取指定用户的关联的机构列表.




```
    async def get_ous_by_user(self, db: AsyncSession, user_id: str) -> list[int]:
        """获取指定用户的关联的机构列表"""
        # 方式一，子查询方式
        stmt = select(User).options(selectinload(User.ous)).where(User.id == user_id)
        result = await db.execute(stmt)
        user = result.scalars().first()
        ous = user.ous if user else []

        # 方式二，关联表方式
        # stmt = (
        #     select(Ou)
        #     .join(user_ou, User.id == user_ou.c.user_id)
        #     .where(user_ou.c.user_id == user_id)
        # )
        # result = await db.execute(stmt)
        # ous = result.scalars().all()

        ouids = [ou.id for ou in ous]
        return ouids
```


上面两种方式是等效的，一个是通过orm关系进行获取关系集合，一个是通过中间表的关系检索主表数据集合。


通过中间表，我们也可以很方便的添加角色的关系，如下面是为角色添加用户，也就是在中间表进行处理即可。




```
    async def add_user(self, db: AsyncSession, role_id: int, user_id: int) -> bool:
        """添加角色-用户关联"""
        stmt = select(user_role).where(
            and_(
                user_role.c.role_id == role_id,
                user_role.c.user_id == user_id,
            )
        )

        if not (await db.execute(stmt)).scalars().first():
            await db.execute(
                user_role.insert().values(role_id=role_id, user_id=user_id)
            )
            await db.commit()
            return True

        return False
```


当然。如果我们不用这种中间表的处理方式，也是可以使用常规多对多关系进行添加处理，不过需要对数据进行多一些检索，也许性能会差一些。




```
    async def add_user(self, db: AsyncSession, ou_id: int, user_id: int) -> bool:
        """给机构添加用户"""
        # 可以使用下面方式，也可以使用中间表方式处理
        # 先判断用户是否存在
        user = await db.get(User, user_id)
        if not user:
            return False

        # 再判断机构是否存在
        result = await db.execute(
            select(Ou).options(selectinload(Ou.users)).filter_by(id=ou_id)
        )
        # await db.get(Ou, ou_id) #这种方式不能获得users，因为配置为selectin
        # await db.get(Ou, ou_id, options=[selectinload(Ou.users)])  # 这种方式可以获得users
        ou = result.scalars().first()
        if not ou:
            return False

        # 再判断用户是否已经存在于机构中
        if user in ou.users:
            return False

        # 加入机构
        ou.users.append(user)
        await db.commit()
        return True
```


 


 本博客参考[楚门加速器p](https://tianchuang88.com)。转载请注明出处！
