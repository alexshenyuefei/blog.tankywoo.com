---
layout: post
title: "Salt 配置多环境"
date: 2015-03-27 14:00
---

在salt的主配置`/etc/salt/master`中有一个例子:

	# The file server works on environments passed to the master, each environment
	# can have multiple root directories, the subdirectories in the multiple file
	# roots cannot match, otherwise the downloaded files will not be able to be
	# reliably ensured. A base environment is required to house the top file.
	# Example:
	# file_roots:
	#   base:
	#     - /srv/salt/
	#   dev:
	#     - /srv/salt/dev/services
	#     - /srv/salt/dev/states
	#   prod:
	#     - /srv/salt/prod/services
	#     - /srv/salt/prod/states

`file_roots` 配置salt配置的存放目录, 其中`base`环境是必要的, 指定`top.sls`存放的位置.

默认没指定环境时则从base目录获取文件

其它则是一些自定义的, 可以通过环境变量指定.

这样可以逻辑上隔离一些环境配置.

每一个环境都可以定义多个目录, 优先级关系由定义目录的顺序决定.

比如:

	file_roots:
	  base:
		- /srv/salt/foo
		- /srv/salt/bar

如果寻找 `salt://file.sls`, 如果都存在`/srv/salt/foo/file.sls`和`/srv/salt/bar/file.sls`, 则使用第一个找到的.

来至 [States tutorial, part 4](http://docs.saltstack.com/en/latest/topics/tutorials/states_pt4.html) 的一个例子:

	file_roots:
	  base:
		- /srv/salt/prod
	  qa:
		- /srv/salt/qa
		- /srv/salt/prod
	  dev:
		- /srv/salt/dev
		- /srv/salt/qa
		- /srv/salt/prod

/srv/salt/prod 里的配置是在三种环境下都可以, /srv/salt/qa 只在qa和dev环境下可用, /srv/salt/dev则只在dev环境下可用.

这样一个配置, 首先在dev环境下测试, 然后进一步移到qa目录下, 做qa检查, 最终在线上prod目录下使用.

另外, 文档是配合pillar使用的, 这块后续再了解. 日常的使用情况偏向客户端使用`salt-call`指定某个具体的state文件来做测试.

如:

	salt-call state.sls path.to.sls test=True

如果定义了`file_roots`的几个环境, 参考 [salt.modules.state.sls](http://docs.saltstack.com/en/latest/ref/modules/all/salt.modules.state.html#salt.modules.state.sls)的手册:

	salt.modules.state.sls(mods, saltenv=None, test=None, exclude=None, queue=False, env=None, **kwargs)

> Specify a file_roots environment.

用于指定某个环境来执行, 默认是None, 会执行base环境.

所以在节点上如果要执行, 则可以改为:

	salt-call state.sls path.to.sls saltenv='dev' test=True

举个最简单例子:

salt master配置:

	master $ more /etc/salt/master.d/common.conf
	file_roots:
	  base:
		- /home/base/
	  dev:
		- /home/dev/
		- /home/base/

base 环境:

	master $ tree /home/base
	/home/base
	├── envtest.sls
	└── top.sls

	master $ more /home/base/envtest.sls
	envtest:
	  cmd.run:
		- name: "echo '[base] env'"

dev 环境:

	master $ tree /home/dev/
	/home/dev/
	├── mytest.sls
	└── top.sls

	master $ more /home/dev/mytest.sls
	envtest:
	  cmd.run:
		- name: "echo '[dev] env'"

客户端执行命令(省略了无用输出):

	minion $ salt-call state.sls mytest saltenv='dev' test=True                                                                                                                                                                                           [16/1464]
			  ID: envtest
		Function: cmd.run
			Name: echo '[dev] env'
		 Comment: Command "echo '[dev] env'" would have been executed

	minion $ salt-call state.sls envtest saltenv='dev' test=True
			  ID: envtest
		Function: cmd.run
			Name: echo '[base] env'
		 Comment: Command "echo '[base] env'" would have been executed

	minion $ salt-call state.sls envtest  test=True
			  ID: envtest
		Function: cmd.run
			Name: echo '[base] env'
		 Comment: Command "echo '[base] env'" would have been executed

可以看到, 指定`saltenv='dev'`时, 会找到mytest.sls并执行.

参考:

* [States tutorial, part 4](http://docs.saltstack.com/en/latest/topics/tutorials/states_pt4.html)
* [File Server Configuration](http://docs.saltstack.com/en/latest/ref/file_server/file_roots.html)
* [自动化运维工具SaltStack - 多环境](http://segmentfault.com/a/1190000000513137)
