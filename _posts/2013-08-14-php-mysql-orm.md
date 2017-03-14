---
layout: post
title: 基于php、mysql实现的对象关系映射
category: 设计
tags: [php, mysql]
---

### 什么是ORM
[ORM](http://en.wikipedia.org/wiki/Object-relational_mapping)的全称是Object Relational Mapping,以面向对象的方式来访问存储在关系型数据库里的信息。

### 简单实现

#### 1、Obj-数据模型抽象

	abstract class Obj
	{
		//对应数据库中列属性
		protected $propTable;
		//对象是否加载，新建则false，从数据库中load则true
		protected $isLoad;

		public function __construct($arg = null)
		{
			if(is_null($arg)) {
				$this->init();
			} else {
				$this->load($arg);
			}
		}

		//魔法方法，访问不存在的属性时系统自动调用此函数
		public function __set($name, $value)
		{
			if(isset($this->propTable[$name])) {
				$this->propTable[$name] = $value;
			} else {
				error_log('field not found!');
			}
		}

		public function __get($name)
		{
			if(isset($this->propTable[$name])) {
				return $this->propTable[$name];
			} else {
				error_log('field not found!');
			}
		}

		public function init()
		{
			$conn = $this->db->getConn();
			/*
			//获取列类型信息
			$sql = 'selecct * from $this->tableName limit 0;';
			$stmt = $conn->prepare($sql);
			$stmt->execute();
			$rs = $stmt->result_metadata(); 
			$arr = array();
			while($field = $rs->fetch_field())
			{
				$arr[$field->name] = $field->def;
			}
			$rs->free_result();
			$stmt->close();
			*/
			//获取列默认值
			$sql = 'desc $this->tableName;'
			$rs = $conn->query($sql); 
			foreach($rs as $val)
			{
				$key = $val['field'];
				$def = $val['default'];
				if(isset($arr[$key])) {
					$arr[$key] = $def;
				}
			}
			$this->propTable = $arr;
			$this->isLoad = false;
		}
	}

#### 2、DBObj-数据库操作类

	abstract class DBObj extends Obj
	{
		protected $tableName;

		protected $db;

		protected $connUrl;

		protected $primaryKey;

		public function __construct($arg)
		{
			$this->db = DB::getInstance($this->connUrl);
			parent::__construct($arg);
		}

		public function load($arg)
		{
			//为简单起见，传入$arg默认为主键
			$sql = "select * from $this->table where $this->primaryKey=$arg;";
			$stmt = $conn->prepare($sql);
			$stmt->execute();
			if(is_null($this->propTable)) {
				$this->init();
			}
			$row = array();
			$keys = array_keys($this->propTable);
			foreach($keys as $key)
			{
				$eval[] = '$row["' . $key . '"]';
			}
			eval('$stmt->bind_result(' . implode(',', $eval) . ');');
			$stmt->fetch();
			foreach($row as $key => $val)
			{
				$this->propTable[$key] = $val;
			}
			$stmt->free_result;
			$this->isLoad = true;
		}

		public function update()
		{
			//将对象信息存入数据库
		}
	}

#### 3、DB-数据库连接类
	
	class DB
	{
		protected $connConfig;

		public function __construct($connUrl)
		{
			$this->connConfig = $this->parseUrl($connUrl);
		}

		public function getConn()
		{
			$conn = mysqli_init();
			@$conn->real_connect(
				$this->connConfig['host'],
				$this->connConfig['user'], 
				$this->connConfig['password'],
				$this->connConfig['database'],
				$this->connConfig['port']
			);
			$conn->query('set names '' . $arg['charset']);
			return $conn;
		}

		public function parseUrl()
		{
		}
	}

### 可以改进的地方

1、由于列属性频繁别访问，可以对整个属性表加以缓存。缓存的实现方式有：类静态变量、xcache缓存

2、对于频繁调用的方法加以缓存。实现方式有：以类名、方法名、方法参数为key、函数返回值为value，缓存于memcache中。

3、update数据入库是对字段进行过滤。
