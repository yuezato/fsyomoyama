# 導入
次のことは捻くれた考え方をしなければまあ成立するのですが
```
(136 * 1024) * 3000 = (4*1024 + 132*1024) * 3000
```
これと似たことをHDDでやるとどうなるかという話をします。
```
time(
3000.times {
  write(136KiB);
})
 =?
time(
3000.times {
 write(4KiB);
 write(132KiB);
}) 
```

https://www.seagate.com/jp/ja/tech-insights/advanced-format-4k-sector-hard-drives-master-ti/
