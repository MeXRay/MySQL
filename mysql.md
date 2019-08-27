# 分数排名
>重点是 distinct用在哪里

```mysql
SELECT *,COUNT(*)+1
FROM Scores s1,
(SELECT DISTINCT(score)
FROM Scores)s
GROUP BY s1.Id HAVING s.score - s1.Score > 0  因为先分组只保留了一个，之后比的时候都只是和一个在比
```
```mysql
SELECT *,COUNT(s.score - s1.Score > 0)+1
FROM Scores s1,
(SELECT DISTINCT(score)
FROM Scores)s
GROUP BY s1.Id  都算上了
```
```mysql
正解
select s.Score,
(select count(distinct Score)
 from Scores
 where score >=s.Score) as Rank
from Scores s 
order by s.Score desc
```
> 结论：不管是having还是where都是发生在select里面的聚合函数之后的，也是在group by之后的。  where来比较我是因为会漏掉最小的一个而选择不用，>=会产生
同等大小的重复，但是他把“一个表当做select之一”实在是大胆，打破思维，这样在里面就ditinct了，之后的比较就真的想人脑想得一样了。我用到了distinct一个表
但是总是把它和Scores表拼接起来，从不敢想过可以用到


# 找出表中连续出现3次的id
> 思路大概：3张表第一张连第二张的第二个id连第三张的id，之后找出这一行3个值相等的数；只通过17/21？  少了个distinct!  这样l1.num连续可能是1112111
1重复了。
```mysql
SELECT l1.`Num` ConsecutiveNums 
FROM LOGS l1,LOGS l2,LOGS l3
WHERE l1.`Id`=l2.`Id`-1 AND l2.`Id`=l3.`Id`-1
AND l1.`Num`=l2.Num AND l2.`Num`=l3.`Num`
```

# 找收入大于经理的员工名
>我先找出ManagerId为NULL的说明他是经理，再把它连接上原来的表  但是正解是：不就是把员工id连接他经理的薪水吗 ，都要连id了，没必要找null！
找null还会出现一个经理不能让两个员工都相连！（真的不够分！）
```mysql
select Name Employee
from Employee ee
join
(select Id,Salary
from Employee 
where ManagerId  = null)e
on ee.ManagerId = e.Id
where ee.Salary > e.Salary
```
```mysql
正解
SELECT e1.Name AS Employee
FROM Employee e1, Employee e2
WHERE e1.ManagerId = e2.Id
	AND e1.Salary > e2.Salary
```
# 寻找部门最高工资员工
> 思路简单：就是把员工表连上部门表，然后根据部门分组求出每组的最高工资，但是下面代码出错了：分组标准错了，不是工资，但是改成部门，每个部门只有1个最高工资，这不对     “所以”需要有一个记录部门最高工资的id表  问题是按部门分组找出最大工资的话，只返回工资，那可能这个工资在我部门不是最高；如果还想返回
id,又只有一个了  “所以”我可以返回工资加“部门名字”！
```mysql
select d.Name Department,e.Name  Employee,Max(e.Salary)  Salary
from Employee e
join Department d on e.DepartmentId = d.Id
group by e.Salary
```
```mysql
我的正解，第一次靠自己的中等
select t.Name Department,ee.Name Employee,t.Salary
from Employee ee
join Department dd on ee.DepartmentId = dd.Id
join
(select max(e.Salary) Salary,d.Name Name
from Employee e
join Department d on e.DepartmentId = d.Id
group by d.Name) t on ee.Salary = t.Salary and dd.Name = t.Name
```
```mysql
看来用where、子查询偶尔挺美观的？  但我join效率高 我自己排版把
select 
    d.Name as Department,
    e.Name as Employee,
    e.Salary 
from 
    Employee e,Department d 
where
    e.DepartmentId=d.id 
    and
    (e.Salary,e.DepartmentId) in (select max(Salary),DepartmentId from Employee group by DepartmentId);
```
#  删除重复的电子邮箱
> distinct无法让没distinct的自动求交集，所以不用distinct，用group by,但是出错，因为要的是“删除”，他这是在清理数据库的表要用delete
```mysql
select *
from Person
where Id in 
(select Id
from Person
group by Email)
```
```mysql
正解：等值连接加排除最小的同邮箱ID
delete p1
from Person p1 ,Person p2
where p1.Email = p2.Email and p1.Id>p2.Id
```
# 日期比较函数用datediff(a,b)=1,既是a-b=1
# update把female和male互换
``` mysql
类似条件判断表达式，我总在想先把一个性别改成另一个性别，这样必须多次update，不合适
update salary
set sex = if(sex = 'm','f','m')
```
# 体育馆连续3天人流量超过100，把那几天都输出（之前遇到类似的题是找出连续3次出现的id,可能存在重复，所以要加distinct）
> 因为有3种情况的要输出，后两天，前后一天和前两天的，所以我想要以一个表为基础去连接其他6个表，把天数连在同一行
```mysql
正解：思路和我一样，只是3个表就够了，要想凑在同一行其实不需要再另外4个表
select distinct(s1.id),s1.visit_date,s1.people
from stadium s1,stadium s2,stadium s3
where s1.people>=100 and s2.people>=100 and s3.people>=100
and 
(
(s2.id =s1.id+1 and s3.id=s1.id+2)or
(s2.id =s1.id+1 and s3.id=s1.id-1)or
(s2.id =s1.id-1 and s3.id=s1.id-2)
)order by s1.id
```

# 求出某几天的出租车下单拒绝率
> 我的思路是找出总的下单数，再找出其中的拒绝数，谁都知道；可是在获得的过程临时表不能复用，一直在建表，和杂乱，最后还语法错误
```mysql
我的错误
select res1.Request_at Day,res2.cancel/res1.total  Cancellation Rate
from
(select t.total,t.Request_at
from
(select t.Request_at,count(*) total
from Trips t,Users u
where t.Id = u.UserId and u.Banned = 'No'
and t.Request_at between '2013-10-01' and '2013-10-03')t
group by t.Request_at)res1
,
select t2.cancel
from
(select t.Status,count(*) cancel
from Trips t,Users u
where t.Id = u.Users_Id and u.Banned = 'No'
and t.Request_at between '2013-10-01' and '2013-10-03')t2
where t2.Status != 1
group by t2.Request_at)res2
(select t.total,t.Request_at
from
(select t.Request_at,count(*) total
from Trips t,Users u
where t.Id = u.Users_Id and u.Banned = 'No'
and t.Request_at between '2013-10-01' and '2013-10-03')t
group by t.Request_at)res1
```
