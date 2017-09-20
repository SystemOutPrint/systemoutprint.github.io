---
layout: post
title:  "PrepareStatement之close"
date:   2017-09-20
categories: MySQL
---

## 0x01 问题

```java
	PreparedStatement stmt = conn.prepareStatement(sql);
	...
	stmt.executeBatch();
	stmt.close();
	conn.commit();
```		

这么写数据无法持久化。

## 0x02 原因

``` java
@Override
public void close() throws SQLException {
	MySQLConnection locallyScopedConn = this.connection;

	if (locallyScopedConn == null) {
		return; // already closed
	}

	synchronized (locallyScopedConn.getConnectionMutex()) {
		if (this.isCached && isPoolable() && !this.isClosed) {
			clearParameters();
			this.isClosed = true;
			this.connection.recachePreparedStatement(this);
			return;
		}

		this.isClosed = false;
		realClose(true, true);
	}
}

// clearParameters内部调用
private void clearParametersInternal(boolean clearServerParameters) throws SQLException {
	boolean hadLongData = false;

	if (this.parameterBindings != null) {
		for (int i = 0; i < this.parameterCount; i++) {
			if ((this.parameterBindings[i] != null) && this.parameterBindings[i].isLongData) {
				hadLongData = true;
			}

			this.parameterBindings[i].reset();
		}
	}

	if (clearServerParameters && hadLongData) {
		// server端也重置该statement
		serverResetStatement();

		this.detectedLongParameterSwitch = false;
	}
}
```
实际上PrepareStatement的close方法会先清空Params，然后再缓存该statement。

## 0x03 解决办法

删除该close语句即可。