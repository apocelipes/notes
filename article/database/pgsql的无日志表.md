没有预写日志（WAL）的表。

创建或者修改现有的表为无日志表：

```sql
CREATE UNLOGGED TABLE foobar (id int);

ALTER TABLE mytable SET UNLOGGED; -- cheap!
ALTER TABLE mytable SET LOGGED; -- expensive!
```

优点：

- 写入很快，差不多是有日志表的2-3倍。
- 占用磁盘空间更小，因为不需要记录WAL。

缺点：

- 崩溃重启后表的所有数据会被清空（正常restart不会清空）
- 只能在主节点访问
- 建不了GiST索引，但可以使用btree和gin。
- 一些备份工具会跳过无日志表。

使用场景：缓存一些只要在主节点访问且丢失了也没关系的数据。

例子，用作缓存：

```golang
type PostgresCache struct {
	db *pgxpool.Pool
}

func NewPostgresCache() (*PostgresCache, error) {
	pgDSN := os.Getenv("POSTGRES_DSN")
	if pgDSN == "" {
		pgDSN = "postgres://user:password@localhost:5432/mydb"
	}

	cfg, err := pgxpool.ParseConfig(pgDSN)
	if err != nil {
		return nil, err
	}

	cfg.MaxConns = 50
	cfg.MinConns = 10

	pool, err := pgxpool.NewWithConfig(context.Background(), cfg)
	if err != nil {
		return nil, err
	}

	_, err = pool.Exec(context.Background(), `
		CREATE UNLOGGED TABLE IF NOT EXISTS cache (
			key VARCHAR(255) PRIMARY KEY,
			value TEXT
		);
	`)
	if err != nil {
		return nil, err
	}

	return &PostgresCache{
		db: pool,
	}, nil
}

func (p *PostgresCache) Get(ctx context.Context, key string) (string, error) {
	var content string
	err := p.db.QueryRow(ctx, `SELECT value FROM cache WHERE key = $1`, key).Scan(&content)
	if err == pgx.ErrNoRows {
		return "", ErrCacheMiss
	}
	if err != nil {
		return "", err
	}
	return content, nil
}

func (p *PostgresCache) Set(ctx context.Context, key string, value string) error {
	_, err := p.db.Exec(ctx, `INSERT INTO cache (key, value) VALUES ($1, $2) ON CONFLICT (key) DO UPDATE SET value = $2`, key, value)
	return err
}
```

性能最多只有redis的70%。
