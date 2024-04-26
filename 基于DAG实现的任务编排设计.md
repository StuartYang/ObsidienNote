# 任务编排工作流


任务编排是什么意思呢，顾名思义就是可以把"任务"这个原子单位按照自己的方式进行编排，任务之间可能互相依赖。复杂一点的编排之后就能形成一个 workflow 工作流了。我们希望这个工作流按照我们编排的方式去执行每个原子 task 任务。例如下图：

![未命名绘图.drawio](https://cdn.jsdelivr.net/gh/StuartYang/oss@master/img/202401081915111.png)

# DAG 有向无环图

每个元素称为顶点 vertex，顶点之间的连线称为边 edge。像我们画的这种带箭头关系的称为有向图，箭头关系之间能形成一个环的成为有环图，反之称为无环图。显然运用在我们任务编排工作流上，最合适的是 DAG 有向无环图。

在代码里存储图有两种数据结构：邻接矩阵和邻接表。

![image-20240108191507528](https://cdn.jsdelivr.net/gh/StuartYang/oss@master/img/202401081915531.png)

![image-20240108191545200](https://cdn.jsdelivr.net/gh/StuartYang/oss@master/img/202401081915830.png)

一般在代码实现上，我们会选择邻接矩阵，这样我们在判断两点之间是否有边更方便点。



## 任务编排的设计

1. 利用DAG的 Node存储任务的先后关系，Edge存储任务的内容和执行状态。
2. 线程池
   1. 执行线程池：从根节点遍历 DAG 结构，运行结束标记节点运行状态和结果
   2. 重试线程池： 周期性查询数据库中是否有任务失败，失败的任务重新执行。失败任务若重新执行，通知执行线程继续完成任务流程的继续进行。

# 任务编排平台化

1. 可拖拽的可视化输入，复杂度更多在前端。
2. 后端：数据的持久化
   1. 将每个节点 Task 的信息给持久化到关系数据库中，包括 Task 的状态、输出结果等。
   2. 数据库来存储各 Task 之间的方向关系。
   3. 在遍历执行 DAG 的整个过程中的中间状态数据持久化到数据库中。

## 数据库设计

- `workflow` 表：来表示一个工作流。

- `task` 表，来表示一个执行单元。用一个`task_parents` 的字段，它是一个 string，存储 parents 的 taskId，多个由分隔符分隔。

  ```sql
  task_id  workflow_id  task_name  task_status  result  task_parents
    1          1           A           0                    -1
    2          1           B           0                    1
    3          1           C           0                    -1
    4          1           D           0                    2,3
  
  ```

  

### 如何遍历

```sql
-- 获取初始化节点 Task A 和 Task C，将其提交到我们的线程池中。
select * from task where workflow_id = 1 and task_parents = -1

-- 对应框架代码中的doExecute(processedNode.getChildren());
-- 得到 Task C 的孩子节点 Task D，这里使用了模糊查询是因为我们的 task_parents 可能是由多个父亲的 taskId 与分隔号组合而成的字符串。查询到孩子节点后，继续提交到线程池即可。  
select * from task where task_parents like %3%;

-- 对应框架代码中的!processedNodes.contains(node) && processedNodes.containsAll(node.getParents())
-- 个数为 0 即保证 parents task 全部成功。
select count(1) from task where task_id in (2,3) and status != 1
```





# 任务的重试机制

重试方案：

增加 task 的重试次数字段` re_try_num`和重试线程池，task 失败后修改数据库中的任务状态和重试次数，让重试线程池定时查询，任务失败或者重试次数未到指定次数（例如 16 次），进行重新调用。



> 原文：https://juejin.cn/post/6911518103402872846
