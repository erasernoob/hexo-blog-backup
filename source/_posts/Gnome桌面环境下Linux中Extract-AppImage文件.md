---
title: Gnome桌面环境下Linux中Extract AppImage文件
date: 2024-09-25 09:37:40
tags: 折腾
---

又是一个感觉浪费了一点时间的地方，效率太低了，虽然学到了一点小东西，还是记录下。

*****

起因是因为这次下定决心想要使用Obsidian进行全面的做笔记了，今晚本来是想要开始使用Obsidian进行美美的打开代码随想录和VScode进行刷题。无奈早就已经出现的那个问题果然就这样在这个时间点进行困扰着我了，第一次使用VScode在Ubuntu中的时候，就发现了，怎么我在Vscode中打不出来优美的汉字了呢，就算切换到了输入法还是不行。然后这个问题就被搁置了，一直在使用我蹩脚的英语~~来写代码的注释~~，还说这样可以提升英语水平来着。然后今天问题就大爆发了，Obsidian竟然也不能在记笔记的时候输入中文，那好吧，硬着头皮用英语记了几节课程笔记~~总有一天会把这个Ubuntu输入法给惩治一顿的~~。好吧，今天妥协了，在用来刷题的笔记那不能用英语写吧，当然最重要的还是我太菜了。

总之，最后就发现，原来是Unbuntu这害人不浅的`snap sotre`原来里面大部分的软件都是进行阉割过的，有一些使用`snap`包来进行安装一些源，也是阉割过的，并没有集成中文的输入法框架等。所以只能通过今天的主角`AppImage`来进行安装完整的软件了。问题又来了，通过修改AppImage文件的权限，添加可运行权限。可运行之后，下一次打开也就只能重新点击AppImage文件，并不是一个真正的应用模式。所以还需要将AppImage文件mount到桌面当中。

得知，原来在Gnome中所有绑定在应用栏中的文件，都有一个`.desktop`文件存放在`/usr/share/applications/` 路径当中，想要将一个AppImage文件注册为一个应用程序，需要按照相应的格式，编写一个`desktopEntry`文件（也就相当于是一个bash脚本）。格式如下：

新建一个`.desktop`文件：（注意以后需要一个特定的AppImage文件夹来存放所有AppImage）

```bash
vim ~/.local/share/applications/obsidian.desktop
```

```bash
[Desktop Entry]
Name=Obsidian
Exec=/home/erasernoob/Downloads/app/squashfs-root/obsidian
Icon=/path/to/obsidian/icon.png  # 你可以替换为实际的图标路径
Type=Application
Categories=Utility;
Terminal=false
```

其中的所有路径和变量的位置，找到又是一个问题。最后发现，是需要`extract`目标的AppImage文件之后得到详细的文件信息。

```bash
 ./your.AppImage --appimage-extract
```

将会得到一个神秘的`squashfs-root`文件夹，里面包含了所有集成在镜像文件中的所有东西，路径也就可以按照此进行编写。

文件编写好之后，需要对桌面文件数据库进行更新：

```bash
update-desktop-database ~/.local/share/applications/
```

更新之后就大功告成了。打开桌面栏，应用已经出现在列表里了。
