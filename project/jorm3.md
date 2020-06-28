# sqlit3 数据库操作的实现与解析

## 设计思想
选择官方c接口，实现Idb通用接口。具体的数据库操作，主要由两个函数ExecQuerySql和ExecNoneQuerySql来封装，底层的操作，主要使用sqlite3_prepare_v2来实现。

## 具体实现

### 数据库连接处理
因为Sqlit3是文件型数据库，所以用智能指针来保存打开数据库的句柄，防止内存泄漏。
```
std::unique_ptr<sqlite3, Deleter> mSQLitePtr;
```
对应的释放结构
```
struct Deleter
{
	void operator()(sqlite3* apSQLite) {
		const int ret = sqlite3_close(apSQLite);
		(void)ret;
		SQLITECPP_ASSERT(SQLITE_OK == ret, "database is locked");
	};
};
```
### 打开数据库连接

```
Sqlit3Db(const char* apFilename,
	const int   aFlags = OPEN_READWRITE,
	const int   aBusyTimeoutMs = 0,
	const char* apVfs = nullptr) : mFilename(apFilename)
{
	sqlite3* handle;
	const int ret = sqlite3_open_v2(apFilename, &handle, aFlags, apVfs);		//打开数据库句柄
	mSQLitePtr.reset(handle);													//保存到智能指针中
	...  																		//错误处理，略
};
```

### 执行返回结果集的查询

```
Rjson ExecQuerySql(string aQuery, vector<string> fields) {
	Rjson rs = Utils::MakeJsonObjectForFuncReturn(STSUCCESS);			//默认返回成功，使用utils.h中的json组装函数
	sqlite3_stmt* stmt = NULL;
	sqlite3* handle = getHandle();										//取得打开的数据库句柄
	string u8Query = Utils::UnicodeToU8(aQuery);
	const int ret = sqlite3_prepare_v2(handle, u8Query.c_str(), static_cast<int>(u8Query.size()), &stmt, NULL);
	if (SQLITE_OK != ret)
	{
		string errmsg = sqlite3_errmsg(getHandle());					//取得错误信息，重置返回信息
		rs.ExtendObject(Utils::MakeJsonObjectForFuncReturn(STDBOPERATEERR, errmsg));
	}
	else {
		... //处理获取表字段名称的sql语句，略
		sqlite3_get_table(handle, aQueryLimit0.c_str(), &pRes, &nRow, &nCol, &pErr);
		for (int j = 0; j < nCol; j++)
		{
			string fs = *(pRes + j);
			if (find(fields.begin(), fields.end(), fs) == fields.end()) {
				fields.push_back(fs);								//保存数据表字段名称
			}
		}
		...
		vector<Rjson> arr;											//存储查询结果集的json对象数组
		while (sqlite3_step(stmt) == SQLITE_ROW) {
			Rjson al;
			for (int j = 0; j < nCol; j++)
			{
				string k = fields.at(j);
				int nType = sqlite3_column_type(stmt, j);			//取得字段值的类型
				if (nType == 1) {					//SQLITE_INTEGER
					al.AddValueInt(k, sqlite3_column_int(stmt, j)); //数值类型处理
				}
				else if (nType == 2) {				//SQLITE_FLOAT
				}
				else if (nType == 3) {				//SQLITE_TEXT    字符串类型处理
					al.AddValueString(k, Utils::U8ToUnicode((char*)sqlite3_column_text(stmt, j)));
				}
				else if (nType == 4) {				//SQLITE_BLOB
				}
				else if (nType == 5) {				//SQLITE_NULL
				}									//暂时只处理了数值与字符串，其它暂未用到
			}
			arr.push_back(al);
		}
		if (arr.empty())							//结果集为空，返回查询空
			rs.ExtendObject(Utils::MakeJsonObjectForFuncReturn(STQUERYEMPTY));
		rs.AddValueObjectArray("data", arr);		//增加结果集，类型为数组
	}
	sqlite3_finalize(stmt);
	cout << "SQL: " << aQuery << endl;
	return rs;
	}
```

### 执行无结果集的查询
```
Rjson ExecNoneQuerySql(string aQuery) {
	Rjson rs = Utils::MakeJsonObjectForFuncReturn(STSUCCESS);
	sqlite3_stmt* stmt = NULL;
	sqlite3* handle = getHandle();
	string u8Query = Utils::UnicodeToU8(aQuery);
	const int ret = sqlite3_prepare_v2(handle, u8Query.c_str(), static_cast<int>(u8Query.size()), &stmt, NULL);
	if (SQLITE_OK != ret)
	{
		string errmsg = sqlite3_errmsg(getHandle());
		rs.ExtendObject(Utils::MakeJsonObjectForFuncReturn(STDBOPERATEERR, errmsg));
	}
	else {
		sqlite3_step(stmt);						//只需要执行一次就好
	}
	sqlite3_finalize(stmt);
	cout << "SQL: " << aQuery << endl;
	return rs;
}
```

### 智能查询实现
提取保留关键字，它们是fuzzy,sort,page,size,sum,count,group，完成排序、分页、分组等特殊查询操作。
```
string fuzzy = params.GetStringValueAndRemove("fuzzy");					//精确匹配与模糊查询切换开关
string sort = params.GetStringValueAndRemove("sort");					//排序
int page = atoi(params.GetStringValueAndRemove("page").c_str());		//分页
int size = atoi(params.GetStringValueAndRemove("size").c_str());		//分页大小
string sum = params.GetStringValueAndRemove("sum");						//字段求和
string count = params.GetStringValueAndRemove("count");					//字段统计
string group = params.GetStringValueAndRemove("group");					//分组
```
处理关键字 ins,lks,ors ，完成in查询，多字段模糊与查询，多字段精确或查询。
```
if (k.compare("ins") == 0) {
	string c = ele.at(0);		//ins参数处理，第一个是字段名，后面是多个查询值
	vector<string>(ele.begin() + 1, ele.end()).swap(ele);	//拼接in查询sql语句
	whereExtra.append(c).append(" in ( ").append(Utils::GetVectorJoinStr(ele)).append(" )");
}
else if (k.compare("lks") == 0 || k.compare("ors") == 0) {
	whereExtra.append(" ( ");
	for (size_t j = 0; j < ele.size(); j += 2) {	//lks与ors参数处理，（字段，值）的重复
		if (j > 0) {
			whereExtra.append(" or ");
		}
		whereExtra.append(ele.at(j)).append(" ");	//拼接sql语句
		string eqStr = k.compare("lks") == 0 ? " like '" : " = '";
		string vsStr = ele.at(j + 1);
		if (k.compare("lks") == 0) {
			vsStr.insert(0, "%");
			vsStr.append("%");
		}
		vsStr.append("'");
		whereExtra.append(eqStr).append(vsStr);
	}
	whereExtra.append(" ) ");
}
```
处理值，不等操作是在值中加入了不等符号；fuzzy开关。
```
if (Utils::FindStartsCharArray(QUERY_UNEQ_OPERS, (char*)v.c_str())) {
	vector<string> vls = Utils::MakeVectorInitFromString(v);
	if (vls.size() == 2) {		//一个不等条件处理
		where.append(k).append(vls.at(0)).append("'").append(vls.at(1)).append("'");
	}
	else if (vls.size() == 4) {	//一对不等条件处理
		where.append(k).append(vls.at(0)).append("'").append(vls.at(1)).append("' and ");
		where.append(k).append(vls.at(2)).append("'").append(vls.at(3)).append("'");
	}
}
else if (!fuzzy.empty() && vType == kStringType) {	//精确查询与模糊查询切换
	where.append(k).append(" like '%").append(v).append("%'");
}
else {
	if (vType == kNumberType)		//数值类型不加单引号
		where.append(k).append(" = ").append(v);
	else							//字符类型要加单引号
		where.append(k).append(" = '").append(v).append("'");
}
```
分页处理
```
if (page > 0) {
	page--;
	querySql.append(" limit ").append(Utils::IntTransToString(page * size)).append(",").append(Utils::IntTransToString(size));
}
```

### 批量插入
多值插入，sqlit3采用标准sql语法，insert into users (field1,field2)  select 'val1','val2' union all  select 'val3','val4' union all  select ...
```
Rjson insertBatch(string tablename, vector<Rjson> elements) {
	string sql = "insert into ";
	string keyStr = " (";
	keyStr.append(Utils::GetVectorJoinStr(elements[0].GetAllKeys())).append(" ) ");		//拼接表字段
	for (size_t i = 0; i < elements.size(); i++) {
		vector<string> keys = elements[i].GetAllKeys();
		string valueStr = " select ";
		for (size_t j = 0; j < keys.size(); j++) {									//组装一条纪录
			valueStr.append("'").append(elements[i][keys[j]]).append("'");
			if (j < keys.size() - 1) {
				valueStr.append(",");
			}
		}
		if (i < elements.size() - 1) {												//准备拼接下一条纪录
			valueStr.append(" union all ");
		}
		keyStr.append(valueStr);
	}
	sql.append(tablename).append(keyStr);
	return ExecNoneQuerySql(sql);											//调用无结果集操作
}
```

### 事务操作
```
Rjson transGo(vector<string> sqls, bool isAsync = false) {
	char* zErrMsg = 0;
	bool isExecSuccess = true; 
	sqlite3_exec(getHandle(), "begin;", 0, 0, &zErrMsg);				//开始事务处理
	for (size_t i = 0; i < sqls.size(); i++) {							
		string u8Query = Utils::UnicodeToU8(sqls[i]);					//处理中文
		int rc = sqlite3_exec(getHandle(), u8Query.c_str(), 0, 0, &zErrMsg);	//处理一个操作
		if (rc != SQLITE_OK)
		{
			isExecSuccess = false;
			cout << "Transaction Fail, sql " << i + 1 << " is wrong. Error: " << zErrMsg << endl;
			sqlite3_free(zErrMsg);
			break;
		}
	}
	if (isExecSuccess)					//成功，提交到数据库
	{
		sqlite3_exec(getHandle(), "commit;", 0, 0, 0);
		sqlite3_close(getHandle());
		cout << "Transaction Success: run " << sqls.size() << " sqls." << endl;
		return Utils::MakeJsonObjectForFuncReturn(STSUCCESS, "insertBatch success.");
	}
	else								//有失败的，回滚
	{
		sqlite3_exec(getHandle(), "rollback;", 0, 0, 0);
		sqlite3_close(getHandle());
		return Utils::MakeJsonObjectForFuncReturn(STDBOPERATEERR, zErrMsg);
	}
}
```

## 项目地址
```
https://github.com/zhoutk/Jorm
```

## 系列文章规划
  - c++操作关系数据库通用接口设计（JSON-ORM c++版）
  - Rjson -- rapidjson代理的设计与实现
  - sqlit3 数据库操作的实现与解析
  - mysql 数据库操作的实现与解析
  - postgres 数据库操作的实现与解析
  - oracle 数据库操作的实现与解析
  - mssql 数据库操作的实现与解析
  - 总结（如果需要的话）

## 感受
批量插入的标准sql竟然很陌生，平时被mysql的方言惯坏了...

