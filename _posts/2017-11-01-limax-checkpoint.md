---
layout: post
title:  "limax-checkpoint"
date:   2017-11-01
author:       "CaiJiahe"
header-img:   "img/tag-bg.jpg"
header-mask:  0.3
catalog:      true
multilingual: true
tags:
    - Limax
    - 缓存
---

```java
void checkpoint() {
    // 尝试加载磁盘备份snapshot
	List snapshotCopy = loadFromDisk();
	
	// 关闭cleaner
	closeLruCleaner();
	
	// marshal + snapshot
	marshal();
	List snapshot = snapshot();
	
	// 合并两次的snapshot
	if (snapshotCopy != null) {
		snapshot = merge(snapshotCopy, snapshot);
	}
	
	// 备份snapshot
	snapshotCopy = snapshot.copy();

	// flush + 重试逻辑
	int retryCnt = 0;
	do {
		retryCnt = flush() ? 0 : retryCnt + 1;
	} while (retryCnt > 0 && retryCnt < N);

	// 如果没重试则打开cleaner，否则根据snapshot的大小来决定是否要进行更激进的操作。
	if (retryCnt >= N) {
		if (snapshotCopy.size() > M) {
			// wtf! shutdown.
		}
		storeToDisk(snapshotCopy);
	} else {
		openLruCleaner();
	}
}
```

* 关闭cleaner的原因是在flush的时候，由于已经释放flush write lock，cleaner可以并发的进行清理数据。所以为了保证数据的一致性，在flush失败的时候先禁用cleaner。但是为了防止OOM的发生，会根据内存数据大小采取failfast等策略。
* 因为从磁盘加载是一个比较耗时的操作，所以建议在flush失败导致数据落到磁盘中的时候置一个内存标志位，表示磁盘文件中有数据。