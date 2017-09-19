---
layout: post
title:  "Spring MVC path小坑"
date:   2017-09-19
categories: Spring
---

## 0x01 问题

		http://localhost:8000/match-queue

对上述URL发送POST请求无法访问到特定Controller，但是使用swagger调试一切正常。		

## 0x02 原因

``` java
	@RequestMapping(value = "/match-room/", method = RequestMethod.POST)
	@ResponseBody
	public Room createMatchRoom(@RequestBody List<Long> roleIds) {
		List<PlayerVo> players = roleIds.stream().map(PlayerVo::new).collect(Collectors.toList());
		return roomService.createMatchRoom(players);
	}
```
代码是这么写的，问题就发生在RequestMapping中的path多写个'/'，所以导致每次都找不到Path，去掉'/'一切正常。<br>
习惯了chrome对斜杠的处理，以为加不加无所谓，但是在Spring中需要完全匹配。