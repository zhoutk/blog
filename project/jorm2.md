# 封装rapidjson用于数据库及网络数据传输

## 背景
我要完成以json为数据媒介，来操作数据库和网络传输。查资料，发现rapidjson是比较流行的json库，并且速度快。但以我的使用方式，用起来非常麻烦，而且我的目的是数据交换。rapidjson非常普通看起来应该是值传输的操作，其实都是内存移动。这虽然能达到高效率的目的，但一不小心就会出错，而且写出来看着非常丑陋，所以我写了个代理类，来达到我的需求。

## 设计思想
只是增加一些方便的操作方法，提供多种构造函数来方便的创建对象，提供复制构造函数，提供方便的增加元素的接口，提供对象的遍历方法等等。内部实际上是一个rapidjson对象，所有的实际操作都在基于rapidjson的，Rjson只是一个代理，改变了使用的方式，提供了值传输，会有一定的性能下降，但这是适应我的需求所作的必要改变。

## 具体实现

### 类结构
Document * json 是实际的json对象，我进行封装，提供新的访问接口，对外不暴露其它细节，比如：Value对象。因此Rjson对象的遍历设计就比较麻烦，最终我选择了类似ES6的方式，先提供一个GetAllKeys方法，再逐一访问每个value的方式来进行遍历。
```
class Rjson {
private:
	Document* json;
public:
	Rjson();
	Rjson(const char* jstr);
	Rjson ExtendObject(Rjson& obj);
	void AddValueInt(string k, int v);
	void AddValueString(string k, string v) ;
	...
}
```

### 默认构造函数

```
Rjson() {
		json = new Document();          //rapidjosn建立Document对象后，必须要增加Value或调用SetObject()来形成一个空json，不然就会报错
		json->SetObject();              //我合并两个操作，建立一个空json
	}
```

### 构造函数，接受char*参数
```
Rjson(const char* jstr) {           //注意这里是const char *，不然从string.c_str()传递过来就要强转，不然会被匹配也string参数的重载构造函数上去。
		json = new Document();      //合并两步操作，简单的处理，可以方便很多创建代码的编写
		json->Parse(jstr);
	}
```

### 构造函数，接受string参数
```
Rjson(string jstr) {            //注意构造函数中调用重载构造函数的方法
		new (this)Rjson(jstr.c_str());
	}
```

### 复制构造函数
```
Rjson(const Rjson& origin) {
		json = new Document();      //使用CopyFrom实现复制
		json->CopyFrom(*(origin.json), json->GetAllocator());
	}
```

### 赋值操作
```
Rjson& operator = (const Rjson& origin) {
		new (this)Rjson(origin);        //利用复制构造函数来实现
		return(*this);
	}
```

### 重载[]运算符
```
string operator[](string key) {     //值都以字符串形式返回
		string rs = "";
		if (json->HasMember(key.c_str())) {
			int vType;
			GetValueAndTypeByKey(key.c_str(), &rs, &vType);
		}
		return rs;
	}
```

### 增加数值类型

```
void AddValueInt(string k, int v) {
		string* newK = new string(k);   //必须新建
		Value aInt(kNumberType);
		aInt.SetInt(v);
		json->AddMember(StringRef(newK->c_str()), aInt, json->GetAllocator());     //addMember方法是地址传递
	}
```
### 增加字符串类型
```
	void AddValueString(string k, string v) {
		string* newK = new string(k);
		Value aStr(kStringType);        //必须新建
		aStr.SetString(v.c_str(), json->GetAllocator());
		json->AddMember(StringRef(newK->c_str()), aStr, json->GetAllocator());
	}
```
### 增加对象数组
```
	void AddValueArray(string k, vector<string>& arr) {
		string* newK = new string(k);
		int len = arr.size();
		Value rows(kArrayType);
		for (int i = 0; i < len; i++) {
			Value al(kStringType);                  //必须新建
			al.SetString(arr.at(i).c_str(),json->GetAllocator());
			rows.PushBack(al, json->GetAllocator());
		}
		json->AddMember(StringRef(newK->c_str()), rows, json->GetAllocator());
	}
```

### 取得所有键

```
vector<string> GetAllKeys() {   
    //不想显露Value对象，使用这个接口加[]运算符来完成遍历，若需要值类型，则与GetValueAndTypeByKey方法配合。
		vector<string> keys;
		for (auto iter = json->MemberBegin(); iter != json->MemberEnd(); ++iter)
		{
			keys.push_back((iter->name).GetString());
		}
		return keys;
	}
```

### 取得指定值及其类型

rapidjson一共定义了七种值类型，需要一一对应处理
```
enum Type {
    kNullType = 0,      //!< null
    kFalseType = 1,     //!< false
    kTrueType = 2,      //!< true
    kObjectType = 3,    //!< object
    kArrayType = 4,     //!< array 
    kStringType = 5,    //!< string
    kNumberType = 6     //!< number
};
```

暂时只处理了数值、字符串及数组类型
```
void GetValueAndTypeByKey(string key, string* v, int* vType) {
		Value::ConstMemberIterator iter = json->FindMember(key.c_str());
		if (iter != json->MemberEnd()) {
			*vType = (int)(iter->value.GetType());
			if (iter->value.IsInt()) {
				std::stringstream s;
				s << iter->value.GetInt();
				*v = s.str();
			}
			else if (iter->value.IsString()) {
				*v = iter->value.GetString();
			}
			else if (iter->value.IsArray()) {
				*v = GetJsonString((Value&)iter->value);
			}
			else {
				*v = "";
			}
		}
		else {
			*vType = kStringType;
			*v = "";
		}
	}
```

### 获取json字符串
```
string GetJsonString() {
		StringBuffer strBuffer;
		Writer<StringBuffer> writer(strBuffer);
		json->Accept(writer);
		return strBuffer.GetString();
	}
```

### 扩展json对象
```
Rjson ExtendObject(Rjson& obj) {
		Document* src = obj.GetOriginRapidJson();
		for (auto iter = src->MemberBegin(); iter != src->MemberEnd(); ++iter)
		{
			if (json->HasMember(iter->name)) {          //键存在，更新
				Value& v = (*json)[iter->name];
				v.CopyFrom(iter->value, json->GetAllocator());
				//v = (Value&)std::move(vTmp);
			}
			else {                                      //键不存在，新增
				string* newK = new string(iter->name.GetString());
				Value vTmp;
				vTmp.CopyFrom(iter->value, json->GetAllocator());
				json->AddMember(StringRef(newK->c_str()), vTmp, json->GetAllocator());
			}
		}
		return *(this);
	}
```

## 使用示例

```
Rjson obj;
obj.AddValueString("username", "插入测试");
obj.AddValueInt("password", 3245);
cout << obj.GetJsonString() << endl;

Rjson obj2(obj);

Rjson obj3("{\"user\":\"bill\",\"age\":12}");
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
这是封装数据库通用访问接口的第一步，提供方便的json对象操作，不然写个json对象得郁闷死。很怀念在javascript中操作json的感觉，如丝般润滑......

