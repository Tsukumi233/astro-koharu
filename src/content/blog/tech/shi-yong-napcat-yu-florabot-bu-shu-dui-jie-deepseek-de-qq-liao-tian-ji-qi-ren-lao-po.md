---
title: 使用NapCat与FloraBot部署对接DeepSeek的QQ聊天机器人（老婆）
link: shi-yong-napcat-yu-florabot-bu-shu-dui-jie-deepseek-de-qq-liao-tian-ji-qi-ren-lao-po
date: 2025-02-10
tags:
  - QQ
  - bot
categories:
  - 技术
---


<h2 id="介绍"><a href="#介绍" class="headerlink" title="介绍"></a>介绍</h2><p>本文用到的开源项目：</p>
<p><a target="_blank" rel="noopener" href="https://github.com/NapNeko/NapCatQQ">NapNeko&#x2F;NapCatQQ: 现代化的基于 NTQQ 的 Bot 协议端实现</a></p>
<p><a target="_blank" rel="noopener" href="https://github.com/FloraBotTeam/FloraBot">FloraBotTeam&#x2F;FloraBot: 一个新的, 使用 Python 编写的支持插件的 ChatBot</a></p>
<p><a target="_blank" rel="noopener" href="https://github.com/umaru-233/My-Dream-Moments/tree/QQ-FloraBotPlugin">umaru-233&#x2F;My-Dream-Moments at QQ-FloraBotPlugin</a></p>
<p>下面记录一下我使用这些项目，在Windows系统部署一个简单的对接DeepSeek API的QQ老婆的过程</p>
<h2 id="流程"><a href="#流程" class="headerlink" title="流程"></a>流程</h2><p>在<a target="_blank" rel="noopener" href="https://github.com/NapNeko/NapCatQQ/releases/">Releases · NapNeko&#x2F;NapCatQQ</a>下载[Win64无头]一键包。解压后运行napcat.bat，根据提示扫码登陆。</p>
<p><img src="https://i.ibb.co/35xzZMjv/image-20250210150629365.png"></p>
<p>之后在终端找到带有token的webui链接。建议修改默认token(密码).</p>
<p>进入你想安装 FloraBot 的目录中, 然后运行以下命令(请使用 PowerShell 来运行):（<a target="_blank" rel="noopener" href="https://github.com/FloraBotTeam/FloraBot-Installer">FloraBotTeam&#x2F;FloraBot-Installer: FloraBot 的一键安装脚本</a>）</p>
<figure class="highlight powershell"><table><tr><td class="gutter"><pre><span class="line">1</span><br></pre></td><td class="code"><pre><span class="line"><span class="built_in">Invoke-WebRequest</span> <span class="literal">-Uri</span> <span class="string">&quot;https://raw.githubusercontent.com/FloraBotTeam/FloraBot-Installer/main/WindowsInstaller.ps1&quot;</span> <span class="literal">-OutFile</span> <span class="string">&quot;WindowsInstaller.ps1&quot;</span>; powershell <span class="operator">-File</span> WindowsInstaller.ps1; <span class="built_in">Remove-Item</span> WindowsInstaller.ps1</span><br></pre></td></tr></table></figure>

<p>如果网络不好，运行：</p>
<figure class="highlight powershell"><table><tr><td class="gutter"><pre><span class="line">1</span><br></pre></td><td class="code"><pre><span class="line"><span class="built_in">Invoke-WebRequest</span> <span class="literal">-Uri</span> <span class="string">&quot;https://github.moeyy.xyz/https://raw.githubusercontent.com/FloraBotTeam/FloraBot-Installer/main/WindowsInstaller.ps1&quot;</span> <span class="literal">-OutFile</span> <span class="string">&quot;WindowsInstaller.ps1&quot;</span>; powershell <span class="operator">-File</span> WindowsInstaller.ps1; <span class="built_in">Remove-Item</span> WindowsInstaller.ps1</span><br></pre></td></tr></table></figure>



<p>根据提示完成后，启动脚本为 Run.bat, 运行该脚本即可启动 FloraBot。配置目录下的<code>Config.json</code>。把<code>ConnectionType</code>的值改为**<code>WebSocket</code>**,这样方便配置.在<code>BotID</code>把0换为机器人QQ号,在<code>Administrator</code>把0换为管理员QQ号,可以写多个.</p>
<p>在<a target="_blank" rel="noopener" href="https://github.com/umaru-233/My-Dream-Moments/tree/QQ-FloraBotPlugin">umaru-233&#x2F;My-Dream-Moments at QQ-FloraBotPlugin</a>点击Code - Download ZIP,解压后的插件放在Plugins目录.最后的目录结构类似:</p>
<figure class="highlight moonscript"><table><tr><td class="gutter"><pre><span class="line">1</span><br></pre></td><td class="code"><pre><span class="line"><span class="name">D</span>:\Apps\FloraBot\FloraBot\Plugins\My-Dream-Moments-QQ-FloraBotPlugin\Main.py...</span><br></pre></td></tr></table></figure>

<p>插件目录有Plugin.json.在DeepSeekApiUrl, DeepSeekApiKey, DeepSeekModel键值对配置自己的api即可.其他配置按需要调整.</p>
<p>回到NapCat的webui.在 网络配置 栏新建一个WebSocket客户端.默认端口和flora一样都是3003不用改动.</p>
<p><img src="https://i.ibb.co/TB551Gk5/image.png"></p>
<p>启动ws客户端后,重新运行florabot的Run.bat.你应该可以看见类似:</p>
<figure class="highlight erlang"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br></pre></td><td class="code"><pre><span class="line">正在加载插件, 请稍后...</span><br><span class="line">正在加载插件 MyDreamMoments ...</span><br><span class="line">框架连接方式为: WebSocket</span><br><span class="line">MyDreamMoments 加载成功</span><br><span class="line">框架已通过 WebSocket 连接, 连接ID: <span class="number">1</span></span><br></pre></td></tr></table></figure>

<p>这样就可以在群聊或者私聊使用命令和机器人聊天了.具体可以发送</p>
<figure class="highlight"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br></pre></td><td class="code"><pre><span class="line">/帮助</span><br><span class="line">/帮助 MyDreamMoments</span><br></pre></td></tr></table></figure>

<p>查看. 群聊内也可以使用@进行聊天.</p>
<p>目前qq插件尚不完善,本文仅供学习参考~</p>
