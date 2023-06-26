# JOIN
> 总结mysql 几种常用的join 类型

## 嵌套循环连接算法（NLJ）
> NLJ 算法一次将一行从外循环传递当内循环，需要多次读取在内循环中处理的表

```
for each row in t1 matching range {                            # 循环t1表匹配范围的每一行
	for each row in t2 matching reference key {                # 循环t2表匹配引用键的每一行
		for each row in t3 {                                   # 循环t3表的每一行
			if row satisfies join conditions, send to client   # 如果该行符合连接条件，发送到客户端
		}
	}
}
```



## 块嵌套循环连接（BNL）
> 块嵌套循环（Block Nested-Loop --> BNL）连接算法采用在外部循环中一次读取一定量的行保存到缓冲区，然后把缓冲的数据全部传递给内部循环，在内部进行处理，从而减少了必须读取内部循环中的表的次数。

+ 在MySQL 8.0.18之前，该算法适用于没有索引的等值连接；在MySQL 8.0.18及更高版本中，在这种情况下，会使用哈希连接优化。从MySQL8.0.20开始，MySQL不再使用块嵌套循环，在以前使用块嵌套循环的所有情况下都使用哈希连接。
如下伪代码表示BNL算法的执行流程：
```
for each row in t1 matching range {                               # 循环t1表匹配范围的每一行
	for each row in t2 matching reference key {                   # 循环t2表匹配引用键的每一行
		store used columns from t1, t2 in join buffer             # 在连接缓冲区中保存t1和t2表中用到的列
		if buffer is full {                                       # 如果缓冲区已满
			for each row in t3 {                                  # 循环t3表的每一行
				for each t1, t2 combination in join buffer {      # 循环缓冲区中每个t1和t2表中列的结合体
				if row satisfies join conditions, send to client  # 如果结合体符合条件，发送到客户端
			}
		}
		empty join buffer                                         # 清空连接缓冲区
		}
	}
}

if buffer is not empty {                    # 如果缓冲区非空（应该是最后一次数据量不大，缓冲区不能满的情形）
	for each row in t3 {
		for each t1, t2 combination in join buffer {
			if row satisfies join conditions, send to client
		}
	}
}

对t3表扫描的次数会随着 join_buffer_size（连接缓冲区大小）值的增加而减少，直到 join_buffer_size足够大，足以容纳所有之前的行组合。
达到这一点后，使它再变大也不能提高速度了。
```

## hash join
>默认情况下，MySQL（8.0.18 及更高版本）尽可能使用散列连接。可以使用BNL和NO_BNL优化器提示之一或通过设置 block_nested_loop=on或 block_nested_loop=off作为优化器开关服务器系统变量设置的一部分 来控制是否使用散列连接 。
>从 MySQL 8.0.20 开始，删除了对块嵌套循环的支持，并且服务器在以前使用块嵌套循环的任何地方都使用哈希连接。
>哈希连接的内存使用可以使用 join_buffer_size系统变量来控制；哈希连接不能使用超过此数量的内存。当散列连接所需的内存超过可用的数量时，MySQL 通过使用磁盘上的文件来处理这个问题。


## 连接缓冲 JOIN buffer
>join buffer 在BNL 算法中发挥了很大的作用
MySQL 连接缓冲具有以下特点：
+ 当连接类型为ALL或INDEX时，可以使用连接缓冲（换句话说，当没有使用possible key时，并且分别对数据行或索引行进行了完整扫描),或range。缓冲的使用也适用于外连接
+ 永远不会为第一个非常量表分配连接缓冲区，即使它的类型是 ALL or INDEX。
+ 只有对连接感兴趣的列存储在其连接缓冲区中，而不是整行。
+ join_buffer_size 系统变量确定用于处理查询的每个连接缓冲区的大小 。
+ 为每个可以缓冲的连接分配一个缓冲区，因此可以使用多个连接缓冲区处理给定的查询。
+ 连接缓冲区在执行连接之前分配，并在查询完成后释放。


## 测试
+ 5.7
  * 无hash join
  * 当被驱动表不存在索引时选择 BNL 算法优化
  * 当被驱动表存在索引时使用 NLJ 使用索引扫描被驱动表
+ 8.0
  * 当被驱动表不存在索引，驱动表存在索引时可能会调换位置，利用索引
  * 当驱动表和被驱动表都不存在索引时会使用hash join
  * 当被驱动表存在索引时使用 NLJ 使用索引扫描被驱动表
  