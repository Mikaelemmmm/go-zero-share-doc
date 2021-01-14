我们在写业务的时候，java、php等其他语言一般分层会把事物放在service层，在go-zero中查了资料，也看了源码也问了go-zero项目组成员，没办法像传统那样去在logic层去begin一下，rollback/commit，这样让我感觉很不爽，只能在model中去开启事务，比如：

我现在用户在注册时候，有user（用户基本信息表）、user_auth（用户认证表），这两个在本地事务肯定要么同时成功要么同时失败，按照go-zero的做法可能就是如下图：

![model trans](/Users/seven/Desktop/go-zero文章/images/七/model trans.jpeg)

这样做可以实现，但是我感觉很不优雅，后来我在想是否可以把事务暴露给logic，在logic中处理呢？于是我就做了一个封装，在每个model中都可以添加一个暴露事务的方法：

![logic trans](/Users/seven/Desktop/go-zero文章/images/七/logic trans.jpeg)

在每个model中跟c、r、u、d一样，同时暴露一个Trans出来，在logic中就可以直接使用事务啦，感觉这样用才爽，不知道你们感觉怎么样呢？ 当然你会说，啊，每次生成完model都要在model中 添加一个Trans方法，重复的工作为什么不交给goctl呢？恭喜你答对了，我们来改造~/.goctl/model/delete.tpl:

```go

func (m *default{{.upperStartCamelObject}}Model) Delete({{.lowerStartCamelPrimaryKey}} {{.dataType}}) error {
	{{if .withCache}}{{if .containsIndexCache}}data, err:=m.FindOne({{.lowerStartCamelPrimaryKey}})
	if err!=nil{
		return err
	}{{end}}

	{{.keys}}
    _, err {{if .containsIndexCache}}={{else}}:={{end}} m.Exec(func(conn sqlx.SqlConn) (result sql.Result, err error) {
		query := fmt.Sprintf("delete from %s where {{.originalPrimaryKey}} = ?", m.table)
		return conn.Exec(query, {{.lowerStartCamelPrimaryKey}})
	}, {{.keyValues}}){{else}}query := fmt.Sprintf("delete from %s where {{.originalPrimaryKey}} = ?", m.table)
		_,err:=m.conn.Exec(query, {{.lowerStartCamelPrimaryKey}}){{end}}
	return err
}

func (m *default{{.upperStartCamelObject}}Model) Trans(fn func(session sqlx.Session)error) error  {
	err := m.Transact(func(session sqlx.Session) error {
		err := fn(session)
		if err != nil{
			return err
		}
		return nil
	})
	return err
}
```

在来生成一下model试试，哈哈 可以了。

至于这里为什么不在~/.goctl/model下单独加一个trans.tpl，是因为目前goctl无法支持，已经提了，可能后面版本会支持，不过这样是曲线救国了，工具嘛，能实现就好了