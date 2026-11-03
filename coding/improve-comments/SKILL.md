---
name: improve-comments
description: 打磨优化中文代码注释，使得其更易于人类阅读，符合个人编写习惯。
---

# improve-comments skill

参考以下内容对代码注释进行优化，主要参考改写方式，具体改写时**仍需遵守 go doc / JSDoc / Javadoc / Python Sphinx 等语言注释标准**。  

## Bad & Good Cases

以下展示均遵循如下格式:  

```
原注释
↓
优化后的注释
```

### 太喜欢使用分号，语气生硬

```
// schema 是内嵌的建表脚本；重复执行必须保持幂等，便于升级旧数据库。
↓
// schema 是数据库初始化脚本，用于创建所需的表和索引，该脚本需要有幂等性
```
> 缓和了语气（更接近**自然中文**），减少不必要的分号使用，末尾不加句号

```
// Store 包装单连接 SQLite 数据库，避免并发写锁竞争。
↓
// Store 包装了单连接的 SQLite 数据库，以避免并发写锁竞争
```
> 缓和了语气（更接近**自然中文**）

```
// SQLite 单写者模型；限制为单连接可避免 "database is locked"。
↓
// 采用 SQLite 单写者模型，将最大连接数限制为 1，减少同一进程内的数据库写锁竞争
```
> 缓和语气，说明清楚缘由和效果。  

### 描述过于简略，措辞不自然

```
// Root 读取唯一同步根配置；从未保存时返回零值且不报错。
func (s *Store) Root() (platform.Root, error) {
	var root platform.Root
	var data []byte
	err := s.db.QueryRow("SELECT data FROM sync_roots WHERE singleton = 1").Scan(&data)
	if errors.Is(err, sql.ErrNoRows) {
		return root, nil
	}
	if err == nil {
		err = json.Unmarshal(data, &root)
	}
	return root, err
}

↓

// Root 会读取唯一的同步根的配置，同步根尚未配置时返回零值但不会报错
...(代码略)
```
> 去掉不合适的分号使用，补充省略的主语“同步根”，措辞“从未保存”改成“尚未配置”，同时优化语气，更靠近自然中文。  

```
// Store 包装单连接 SQLite 数据库，避免并发写锁竞争。
↓
// Store 封装了 SQLite 数据库，并通过单连接限制来减少并发写入时的锁冲突
```
> 写清楚了是怎么避免的，让语气变得更自然。

### 语态混杂，缺少明确的操作对象指代和操作说明，把实现细节包装成抽象概念

```
// Children 返回目录的本地快照，以及该目录是否曾完整枚举过。
// complete=false 时调用方不能把空列表当成“远端确实为空”。
func (s *Store) Children(parent string) ([]platform.Entry, bool, error) {
	var complete int
	err := s.db.QueryRow("SELECT count(*) FROM snapshots WHERE parent_id = ?", parent).Scan(&complete)
	if err != nil {
		return nil, false, err
	}
	rows, err := s.db.Query("SELECT data FROM nodes WHERE parent_id = ? ORDER BY path", parent)
	if err != nil {
		return nil, false, err
	}
	defer rows.Close()
	entries := []platform.Entry{}
	for rows.Next() {
		var data []byte
		var entry platform.Entry
		if err := rows.Scan(&data); err != nil {
			return nil, false, err
		}
		if err := json.Unmarshal(data, &entry); err != nil {
			return nil, false, err
		}
		entries = append(entries, entry)
	}
	return entries, complete == 1, rows.Err()
}

↓

// Children 从数据库中读取了 parent 所指目录的直接子项，并返回该目录是否已被枚举过
// 如果尚未完成过枚举 (complete=false)，即使返回的子项列表为空，也不能据此判断远端目录为空
...(代码略)
```
> 明确指出是“从数据库中读”，并且说清楚读出的内容是什么，而不是概括成本地快照。另外优化语气，补充遗漏的被动语态，贴近自然中文。


### 和代码没有关系的注释

```
// Commit 提交一次目录枚举结果并返回该目录的完整快照。
// 当前实现只添加新条目：已有条目不会因远端版本变化、删除或类型变化而被替换。
// ponytail: 版本对账与删除墓碑属于 M3；提前实现只会增加误删风险。
func (s *Store) Commit(parent string, entries []platform.Entry) ([]platform.Entry, error) {
	old, _, err := s.Children(parent)
	if err != nil {
		return nil, err
	}
	byPath := map[string]platform.Entry{} // 以大写名称索引，匹配 Windows 不区分大小写
	for _, e := range old {
		byPath[strings.ToUpper(e.Name)] = e
	}
	tx, err := s.db.Begin()
	if err != nil {
		return nil, err
	}
	defer tx.Rollback()
	for _, e := range entries {
		if prev, exists := byPath[strings.ToUpper(e.Name)]; exists {
			if prev.Name != e.Name || prev.IsDir != e.IsDir {
				return nil, fmt.Errorf("名称与已有本地条目冲突: %q", e.Name)
			}
			continue
		}
		data, err := json.Marshal(e)
		if err != nil {
			return nil, err
		}
		if _, err = tx.Exec("INSERT INTO nodes(id, parent_id, path, data) VALUES(?, ?, ?, ?)", e.ID, parent, e.Path, data); err != nil {
			return nil, err
		}
	}
	// 只有整批条目都成功写入后才标记目录已完整填充。
	if _, err = tx.Exec("INSERT INTO snapshots(parent_id, populated_at) VALUES(?, datetime('now')) ON CONFLICT(parent_id) DO UPDATE SET populated_at=excluded.populated_at", parent); err != nil {
		return nil, err
	}
	if err = tx.Commit(); err != nil {
		return nil, err
	}
	result, _, err := s.Children(parent)
	return result, err
}

↓

// Commit 会将 parent 所指目录的枚举结果保存到数据库，并返回该目录已保存的全部直接子项
// 当前仅会保存新增的目录项，不会覆盖已有条目，也不会删除本次枚举中缺失的条目
// 目录项的版本更新和删除处理尚未实现，以避免误删或覆盖已有的本地数据
...(代码略)
```
> 除了符合上文的优化外，去除了和代码无关的 "ponytail: ..." 注释

