## 合并多个mp4

需要先转成ts，再合并：

```bash
for file in *.mp4
do
    ffmpeg -i $file -c:a copy -c:v copy "${file}.ts"
done
```

制作合并需要用的合并清单：

```bash
for file in *.ts
do
    echo "file ${file}.ts" >> list.txt
done
```

合并：

```bash
ffmpeg -f concat -i list.txt -c:a copy -c:v copy output.mp4
```

不借助文件清单合并：

```bash
ffmpeg -i "concat:01.ts|02.ts|03.ts|04.ts" -c:a copy -c:v copy output.mp4
```

## 视频切片

```bash
ffmpeg -ss 00:00:00.000 -t 00:01:11.000 -i test.mp4 -c:a copy -c:v copy cut1.mp4 
```

## 从视频里截图

```bash
ffmpeg -i test.mp4 -y -f mjpeg -ss 3 -s '800x600' -t 1  test1.jpg
ffmpeg -i test.mp4 -y -f image2 -ss 3 -vframes 4 f-%4d.jpg # 多个截图按格式命名
ffmpeg -i out.mp4 -t 10 -pix_fmt rgb24 out.gif

# 每隔20秒截一张图
ffmpeg -i out.mp4 -f image2 -vf fps=fps=1/20 out%d.png

# 每一帧都转成图片
ffmpeg -i out.mp4 out%10d.png

# -t 截图时长，秒或hh:mm:ss.xxx
# -vframes 截多少帧
# -s 图片宽x高
```
