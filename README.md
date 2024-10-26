# Readme
A note about Network-Based Locking System (NLS).

### Network-Based Locking System (NLS)

当位于不同机器上的应用程序并发地读写一组共享资源时，需要使用NLS。

可以基于database来实现NLS，这种方式not very graceful，but very efficient。

##### Mutex

当访问共享资源的应用程序都是写者的时候，使用Mutex很方便。

Database有一种特性，即创建unique记录的操作是互斥的，可以利用这种特性来实现Mutex。

基于Apache Cassandra实现Mutex的原理如下：

```python
# Prepare schema and table for Mutexes:
session.execute(
  """CREATE TABLE mutexes (
    PRIMARY KEY (resource_type, resource_id),
    resource_type VARCHAR,
    resource_id INT,
    mark VARCHAR,
    acquired_at TIMESTAMP
  );"""
)
```

```python
# Acquire Mutex:
session.execute(
  'INSERT INTO mutexes (resource_type, resource_id, mark, acquired_at) VALUES (%s, %s, %s, toTimestamp(now())) IF NOT EXISTS;',
  ['foobar', 123, 'fszey']
)
```

```python
# Release Mutex:
session.execute(
  'DELETE FROM mutexes WHERE resource_type=%s AND resource_id=%s AND mark=%s;',
  ['foobar', 123, 'fszey']
)
```

基于Redis实现Mutex的原理如下：

```python
# Acquire Mutex:
r.set('mutexes/foobar,123', 'gluww,1729837899653', nx=True)
```

```python
# Release Mutex:
lua_script = \
  """if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
  else
    return 0
  end"""
r.eval(lua_script, 1, 'mutexes/foobar,123', 'gluww,1729837899653')
```

基于Oracle Database或Oracle In-Memory Database实现Mutex的原理如下：

```python
# Prepare schema and table for Mutexes:
cursor.execute(
  """CREATE TABLE mutexes (
    PRIMARY KEY (resource_type, resource_id),
    resource_type CHAR(25) NOT NULL,
    resource_id INTEGER NOT NULL,
    mark CHAR(7) NOT NULL,
    acquired_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP NOT NULL
  );"""
)
```

```python
# Acquire Mutex:
cursor.execute(
  'INSERT INTO mutexes (resource_type, resource_id, mark) VALUES (:resource_type, :resource_id, :mark);',
  ['foobar', 123, 'auykg']
)
```

```python
# Release Mutex:
cursor.execute(
  'DELETE FROM mutexes WHERE resource_type=:resource_type AND resource_id=:resource_id AND mark=:mark;',
  ['foobar', 123, 'auykg']
)
```

注意Acquire Mutex的操作是非阻塞函数调用，一次调用一般不能成功Acquire Mutex，所以在具体实现的时候，需要增加重试机制、延迟重试机制和退出重试机制。

注意附加的两个字段，使用随机生成的`mark`以防止release了其他写者acquired的Mutex，使用`acquired_at`查找因异常情况导致的长期未释放的Mutex。

注意不要试图给Mutex设置超时，因为NLS没有有效的手段阻止已经超时的应用程序继续访问对应的共享资源，如果是因为网络故障而导致锁未被及时释放，应该先修复网络，如果是因为应用程序崩溃而导致的锁未被正常释放，应该先修复应用程序。

### Credits
- Computer Systems: A Programmer's Perspective, Third Edition
- [Readers–writers problem - Wikipedia](https://en.wikipedia.org/wiki/Readers-writers_problem)
- [Readers–writer lock - Wikipedia](https://en.wikipedia.org/wiki/Readers–writer_lock)
- [INSERT IF NOT EXISTS - Apache Cassandra](https://cassandra.apache.org/doc/latest/cassandra/developing/cql/dml.html#insert-statement) and [datastax/python-driver - GitHub](https://github.com/datastax/python-driver)
- [SET NX - Redis](https://redis.io/docs/latest/commands/set/) and [redis/redis-py - GitHub](https://github.com/redis/redis-py)
- [PRIMARY KEY - Oracle](https://docs.oracle.com/en/database/oracle/oracle-database/23/sqlrf/constraint.html) and [oracle/python-oracledb - GitHub](https://github.com/oracle/python-oracledb/)
