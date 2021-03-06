---
layout: post
title: 配置vim+golang开发环境
categories:
- 技术
tags:
- vim 
- golang
---
>一直在使用Subline Text 2写go代码，但是一直想配置个vim+golang的开发环境，没有VIM这个环境就是很不舒服，网上看了挺多的方式，但是感觉每个博客或者回答都是雷同的，大部分都是直接A copy B。总感觉他们都没有自己真正的实践过方法是否可行，或者试过了但是没有将解决问题的关键点写出来，比如在配置的过程中会遇到什么问题等。

##环境需求
要配置使用vim写golang代码，首先机子上要有golang的环境，这部分具体怎么搞网上非常的多，按着教程都不会有问题，我在这里就说说自己在配置vim开发的时候遇到的一些问题。我的机子上的系统是ubuntu 14.04,go 版本为1.3。

###安装插件
vim的配置都知道需要安装某些插件还有写配置文件

######插件管理

关于vim的插件管理的工具网上看到就两种比较多(不知道有没有其它比较好用的),Pathogen和Vundle,这两个插件我都用过，以前是用Pathogen，现在是用Vundle，这个就看哪个比较习惯了。关于这两种插件的用法网上的资料也非常的多，这里也就不罗嗦了。我这里使用的是Vundle这个插件,并且也是看这个[博客](http://tonybai.com/2014/11/07/golang-development-environment-for-vim/)配置的。

######插件
有了插件管理工具，那就可以安装插件了。我是首先直接把vimrc的配置文件写好的，这里我就贴出我的配置文件吧

	1 set cursorline
	2 set encoding=utf-8
  	3 set number
  	4 set nocompatible              " be iMproved, required
  	5 filetype off                  " required
  	6 "colorscheme molokai
  	7 
  	8 " set the runtime path to include Vundle and initialize
  	9 set rtp+=~/.vim/bundle/Vundle.vim
 	10 call vundle#begin()
 	11 
 	12 " let Vundle manage Vundle, required
 	13 Plugin 'gmarik/Vundle.vim'
 	14 Plugin 'fatih/vim-go'
 	15 Plugin 'Valloric/YouCompleteMe'
 	16 Plugin 'Lokaltog/vim-powerline'
 	17 Plugin 'SirVer/ultisnips'
	18 
 	19 " All of your Plugins must be added before the following line
 	20 call vundle#end()            " required
 	21 filetype plugin indent on    " required
 	22 
 	23 " set mapleader
 	24 let mapleader = ","
 	25 
 	26 " vim-go custom mappings
 	27 au FileType go nmap <Leader>s <Plug>(go-implements)
 	28 au FileType go nmap <Leader>i <Plug>(go-info)
 	29 au FileType go nmap <Leader>gd <Plug>(go-doc)
 	30 au FileType go nmap <Leader>gv <Plug>(go-doc-vertical)
 	31 au FileType go nmap <leader>r <Plug>(go-run)
 	32 au FileType go nmap <leader>b <Plug>(go-build)
 	33 au FileType go nmap <leader>t <Plug>(go-test)                     
 	34 au FileType go nmap <leader>c <Plug>(go-coverage)
 	35 au FileType go nmap <Leader>ds <Plug>(go-def-split)
	36 au FileType go nmap <Leader>dv <Plug>(go-def-vertical)
 	37 au FileType go nmap <Leader>dt <Plug>(go-def-tab)
 	38 au FileType go nmap <Leader>e <Plug>(go-rename)
 	39 
 	40 "vim-powerline setting
 	41 let g:Powerline_symbols='fancy'
 	42 set laststatus=2
 	43 
 	44 " vim-go settings
 	45 let g:go_fmt_command = "goimports"
 	46 
 	47 " YCM settings
 	48 let g:ycm_key_list_select_completion = ['', '']
 	49 let g:ycm_key_list_previous_completion = ['', '']
 	50 let g:ycm_key_invoke_completion = '<C-Space>'
 	51                                                                   
 	52 " UltiSnips settings
 	53 let g:UltiSnipsExpandTrigger="<tab>"
	54 let g:UltiSnipsJumpForwardTrigger="<c-b>"
 	55 let g:UltiSnipsJumpBackwardTrigger="<c-z>"
 	56 
 	57 set ts=4 sts=4 sw=4
主要的配置是从`8 " set the runtime path to include Vundle and initialize`这个地方开始的，Vundle安装插件只需要在vimrc这个文件里面写上`Plugin XXX`就可以了，在看了Vundle官网之后发现这个是对于插件代码托管在github上面使用的，非托管在这上面的插件就要写上地址的路径了，详细内容可以查看官网。在安装vim-go的时候会检测你`GOBIN`中是否有`gocode`，`goimports`等等这些工具，没有的话它就会自动下载安装,但是由于众所周知的原因，很多工具的安装会失败，所以对于那些工具我们要进行手动安装，可以自己翻墙去下载也可以到[gopm](http://gopm.io/)这里下载,下载好来后直接使用`go build`生成可执行文件，然后copy到你的`GOBIN`中。上面配置文件中的`Plugin 'Lokaltog/vim-powerline'`跟这里的配置没关系，安装它之后当用vim打开一个文件的使用在最下面会有一个显示栏，显示一些文件的信息。

###遇到的问题
整个安装过程就是这样子，期间也遇到来几个问题导致一直安装不成功，第一个在刚才上面提到的博客地址里也有说到，就是在安装好`YCM`的时候会有这么一段提示

	ycm_client_support.[so|pyd|dll] and ycm_core.[so|pyd|dll] not detected; you need to compile YCM before using it. Read the docs!
解决方法原博客里也有说

	sudo apt-get install build-essential cmake python-dev
	cd ~/.vim/bundle/YouCompleteMe
	./install.sh
第二问题就是我在安装插件的时候一直没有出现文中所说的

	Vundle.vim会在左侧打开一个Vundle Installer Preview子窗口，窗口下方会提示：“Processing 'fatih/vim-go'”，待安装完毕后，提示信息变 成“Done!”
这个现象，后来我把其它插件都先给注释了，单单就安装Vundle这个插件，就好了，估计是之前没有安装这个插件，以为只要执行

	mkdir ~/.vim/bundle
	git clone https://github.com/gmarik/Vundle.vim.git
就好了，其实还是要再vim中执行一次插件安装的,毕竟它也是插件，管理插件的插件，^_^。这个安装是在vim内使用`：PluginInstall`安装的，这下安装好了，就出现来原文所说的显示，心情也好了很多。然后就很顺利的安装了其它的插件,这里`godef`我在下载来的`code.google.com/p/go.tools`包中没找到，所以就先略过来，可能是我没看见。不过问题不大。下面图片就是我安装好之后界面

![alt](/image/2014blog/20141113.png)

##无关的话
前天吧，我将ubuntu桌面切换到了`gnome-classic`，有个`cairo-dock`，但是问题来了，之前用的很开的`alt+tab`组合键切换窗口失效了，然后就去网上找解决方法,总之找了很多中文的一些论坛或者回答的解决方法，但是答案就只有一个，并且连文字都一模一样，包括在哪里加标点，感到十分的惊人。但是他们说的这一个方法也没有解决这个问题。我也不知道那些回答的人到底自己试过没有，估计就是copy一族。本想着再找不到就去`stackoverflow`上直接问了，后来在`google`上搜索到了答案，是一个国外的论坛，答案一针见血，十分开心。可能也是我对这些配置不熟悉的缘故吧。
安装Compizconfig setting manage工具，然后打开选中窗口管理，把shift switcher勾选上就OK了，之前的方法一直都是说把Static Application Switcher勾上，所以一直没效果，其实这两个选项是紧靠在一起的，可我每次勾它身边的人却不认识它。
