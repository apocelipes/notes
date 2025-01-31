可选工具，不包括OBS这种重量级录屏工具或者ffmpeg这种比较底层的工具：

- [peek](https://github.com/phw/peek): 项目标记为废弃，不支持wayland，开发进度停滞。
- [simplescreenrecorder](https://github.com/MaartenBaert/ssr): 功能很全，但只能录成视频再转gif，开发活跃但不支持wayland。
- <gifcap.dev>: 利用WebRTC录制屏幕，需要自己把格式转换成gif，有浏览器就能用，而且可以做成离线应用，wayland上能用，选择自定义区域时不太灵活。
- [kooha](https://github.com/SeaDve/Kooha): 功能最全的一个，支持多种格式，wayland下可用但自定义选择区域一样有bug。
- spectacle，kde自带的，能在wayland上正常使用没有bug，但格式需要转换，而且参数没法调整。
- [vokoscreenNG](https://github.com/vkohaupt/vokoscreenNG): 也能在wayland上正常使用，参数可调，还自带mkv转gif工具，但窗口录屏开始和选择区域时不会自行隐藏，快捷键很难用且在窗口最小化后无法生效。

优化gif，只能减小一点大小：

```bash
# webm转gif
magick a.webm b.gif
# lossy 取值30-200，一般选80可以在不大幅度牺牲画质的前提下减小文件大小，根据图片内容，画面简单纯色区域多的图片能减小一半左右
# https://ezgif.com/optimize 使用了类似的做法，尺寸减小幅度类似
gifsicle -i input.gif -O3 --colors 256 --lossy=80 -o output.gif
```
