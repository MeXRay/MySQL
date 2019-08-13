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
