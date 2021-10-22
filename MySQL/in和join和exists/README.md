# join
以 **select * from ta inner join tb on ta.user_id = tb.user_id** 为例
### 一、 Simple Nested-Loop Join（简单的嵌套循环连接）
<img src="https://raw.githubusercontent.com/dark-tone/notes/main/MySQL/imgs/2.jpg" weight="463" height="418"><br>
相当于两层for循环，比较次数 = ta的行数 * tb的行数。

### 二、 Index Nested-Loop Join（索引嵌套循环连接）

内层表的匹配字段拥有索引，可直接根据索引来进行关联。
### 三、 Block Nested-Loop Join（缓存块嵌套循环连接）
<img src="https://raw.githubusercontent.com/dark-tone/notes/main/MySQL/imgs/3.jpg" weight="580" height="390"><br>
简单的嵌套循环查询，左表的每一条记录都会对右表进行一次扫表，扫表的过程其实也就是从内存读取数据的过程，这个过程其实是比较消耗性能的。所以缓存块嵌套循环连接算法意在通过一次性缓存外层表的多条数据，以此来减少内层表的扫表次数，从而达到提升性能的目的。如果无法使用Index Nested-Loop Join的时候，数据库是默认使用的是Block Nested-Loop Join算法的。
>- 使用Block Nested-Loop Join 算法需要开启优化器管理配置的optimizer_switch的设置block_nested_loop为on 默认为开启，如果关闭则使用Simple Nested-Loop Join 算法。
>- 通过join_buffer_size参数可设置join buffer的大小。(尽量减少不必要的字段查询，字段越少，join buffer所缓存的数据就越多）

# in
in 是把外表和内表作hash 连接，in()只执行一次，它查出B表中的所有id字段并缓存起来。之后，检查A表的id是否与B表中的id相等，如果相等则将A表的记录加入结果集中，直到遍历完A表的所有记录。

IN()适合B表比A表数据小的情况。

# exists
exists()会执行A.length次，它并不缓存exists()结果集，因为exists()结果集的内容并不重要，重要的是其内查询语句的结果集空或者非空，空则返回false，非空则返回true。

EXISTS()适合B表比A表数据大的情况。
# 参考文章
[Mysql Join算法原理](https://zhuanlan.zhihu.com/p/54275505)