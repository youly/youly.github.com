---
layout: post
title: 一个简单的php-mvc框架实现
category: 设计
tag: [php]
---


###什么是MVC
***
[维基介绍](http://en.wikipedia.org/wiki/Model-view-controller)

MVC是软件工程中的一种架构模式，成功地使用这种模式能够将复杂的业务逻辑和视图分离开，不仅能简化编码复杂度，还能减少代码重复量，提高编程效率。

* M-Model， 处理数据库逻辑，关系实体映射。

* V-View， 处理与用户直接相关的界面逻辑。

* C-Controller， 处理具体的业务逻辑。

在实际的MVC框架中可能还会有Action，在调用controller的方法之前由它先处理权限、Cookie等问题。controller在这种设计中可以看成java的类库，封装一些业务逻辑或算法供action调用。

###实现一个简单的框架
***
####目录结构与命名规范

首先框架应该有一个良好的目录结构以及命名规范。

![目录结构](/assets/images/directory.png)

上图是参考rails的目录结构稍作了点修改，去掉了那些我觉得暂时还不需要的，每个目录的作用如下：

* app-具体应用程序的逻辑。    

* config-服务器/数据库/缓存等的具体配置    

* db-数据库的备份     

* lib-框架代码    

* public-静态HTML页面及一些css、images、js等     

* script-存放一些后台执行的命令行脚本     

* tmp-存放一些临时性的文件和数据 

命名规范我觉得rails的Convention over configuration做的相当完美，我们也遵循这个原则：

1. 所有的model类似XxxModel.php(第一个字母大写)，在数据库中相应的表名为xxxs（小写、复数）。

2. 所有的controller类似XxxsController.php（第一个字母大写、复数）。

3. 所有的view类似xxxs/view.php。

例如网站用户(user)，其对应的mvc分别为UserModel,views/users/profile.php,UsersController。表名为users。

####Web服务器配置
如果我们使用apache的虚拟主机，配置如下：

	<VirtualHost *:80>
		ServerName youly.com
		RewriteEngine On
		RewriteRule ^/(.*)$ http://www.youly.com/$1 [L,R]
	</VirtualHost>

	<VirtualHost *:80>
	    ServerName www.youly.com
	    ServerAlias *.youly.com
	    DocumentRoot /path/to/public
	    <Directory "/path/to/public">
	    </Directory>
	    RewriteEngine on
	    RewriteRule ^/(assets\/.*)$ /$1 [L] 
	    RewriteRule ^/(.*)$ /index.php [L] 
	</VirtualHost>

上面的重写规则指定除了public目录之外所有的请求都重定向到public目录下的index.php文件。可[点此](http://www.yourhtmlsource.com/sitemanagement/urlrewriting.html)了解apache的重写规则。打开index.php文件，我们可以看到如下的两行代码：

	Dispatcher::getInstance()->dispatch();

这个文件很简短，只有两行。注意它并没有以?>结尾，主要是为了避免在我们的输出中注入多余的空格。那么Dispatcher是什么？它来自哪里？

####php配置
ROOT为网站根目录。打开php.ini并设置auto_prepend_file=ROOT/config/prepend.php，这样每次执行用户请求的php文件前php内核将先执行prepend.php。打开prepend.php文件，我们可以看到：

	define('DEVELOPMENT_ENVIRONMENT', true);
	define('DS', DIRECTORY_SEPARATOR);
	define('ROOT', '/Users/meituan-it/dev/php-mvc');
	define('APP_PATH', ROOT . DS . 'app');
	define('URI', isset($_SERVER['REQUEST_URI']) ? urldecode($_SERVER['REQUEST_URI']) : '');
	define('DB_HOST', '127.0.0.1:3306');
	define('DB_USER','root');
	define('DB_PASSWORD','123');
	define('DB_NAME','youly');

	set_include_path(ROOT . DS . 'lib' . PATH_SEPARATOR . get_include_path());

	function __autoload($class) {
	    $path = sprintf('%s/lib/%s.php', ROOT, $class);
		if(class_exists($class) || interface_exists($class)) {
	        return true;
	    } else if(file_exists($path)) {
	        include_once($path);
	        return true;
	    } else if(substr($class, -5) == 'Model') {
	        $path = sprintf('%s/models/%s.php', APP_PATH, $class);
		} else if(substr($class, -10) == 'Controller') {
			$path = sprintf('%s/controllers/%s.php', APP_PATH, $class);
		} else if (substr($class, -6) == 'Helper') {
			$path = sprintf('%s/helpers/%s.php', APP_PATH, $class);
	    }
	    if(file_exists($path)) {
	        include_once($path);
	        return true;
	    }
		return false;
	}

	function setReporting() {
		if (DEVELOPMENT_ENVIRONMENT == true) {
		    error_reporting(E_ALL);
		    ini_set('display_errors','On');
		} else {
		    error_reporting(E_ALL);
		    ini_set('display_errors','Off');
		    ini_set('log_errors', 'On');
		    ini_set('error_log', ROOT.DS.'tmp'.DS.'logs'.DS.'error.log');
		}
	}

	setReporting();


这里重写了php原有的__autoload函数，使得它能自动加载我们自己定义的类。setReporting函数是为了方便我们调试。

####第一个类

我们的第一个类Dispatcher，负责分发从前端传来的请求。打开lib/Dispatcher.php文件：

	class Dispatcher
	{
	    protected static $front;
	    const URI_REGEXP = '/^\/([^\d\?\/\\][^\/]*?)?(?:\/([^\d\?\/\\][^\/]*?))?(?:\/([^\?]+?))?(?:\/\?.*)?$/s';

		public static function getInstance()
		{
			if(!isset(self::$front))
			{
				self::$front = new self();
			}
			return self::$front;
		}

		public static function dispatch()
	    {
	        if(!preg_match(self::URI_REGEXP, URI, $urlArray)) {
	            return BaseController::page404();
	        }
	        $class = isset($urlArray[1]) && !empty($urlArray[1]) ? $urlArray[1] : 'index';
	        $action = isset($urlArray[2]) && !empty($urlArray[2]) ? $urlArray[2] : 'default';
	        if(isset($urlArray[3])) {
	            $arr = explode('/', $urlArray[3]);
	            foreach($arr as $k => $v) {
	                $arr[$k] = trim(urldecode($v));
	            }
	            $args = $arr;
	        } else {
	            $args = array();
	        }
	        //var_dump($controllerName, $action, $urlArray, $args);
	        $method = ucwords($action);
			if(strcasecmp($_SERVER['REQUEST_METHOD'], 'POST') === 0) {
				$method = 'actPost' . $action;
			} else {
				$method = 'act' . $action;
			}

			$controller = ucwords($class) . 'Controller';
	        $model = ucwords($class) . 'Model';
	        $view = $class;
			$controllerObj = new $controller($model, $view, $action);

			if ((int)method_exists($controller, $method)) {
				call_user_func_array(array($controllerObj, $method), $args);
				$controllerObj->renderTemplate();
			} else {
				return BaseController::page404();
			}
		}
	}


这个类的主要函数是dispatch，由它来负责分发请求。处理的url类似 **youly.com/controllerName/actionName/queryString**。关于这个正则表达式的分析看[这篇文章](/2013/07/18/php-regular-expression-greedy-or-nongreedy)。根据url中的控制器和动作，框架构造一个继承自BaseController的子类来处理具体的请求。

####Controller

打来文件lib/BaseController.php:

	class BaseController {
	     
	    protected $_model;
	    protected $_action;
	    protected $_view;
	 
	    public function __construct($model, $view, $action) {
	         
	        $this->_action = $action;
	        $this->_model = $model;
	        $this->_view = new View($view,$action);
	    }
	 
	    public function set($name,$value) {
	        $this->_view->set($name,$value);
	    }
	 
	    public static function page404()
	    {
		    echo '404 NOT FOUND!';
		    return false;
	    }
	    
	    public function renderTemplate()
	    {
		    $this->_view->render();
	    }
	}

Controller负责从Mode里取数据，并通过视图呈现出来。

####View

	class View {
	     
	    protected $variables = array();
	    protected $_view;
	    protected $_action;
	     
	    function __construct($view,$action) {
	        $this->_view = $view;
	        $this->_action = $action;
	    }
	    //模板变量 
	    function set($name,$value) {
	        $this->variables[$name] = $value;
	    }
	 
	    function render() {
	        extract($this->variables);
	        if(file_exists(APP_PATH . DS . 'views' . DS . 'common' . DS . 'header.php')) {
	        	include (APP_PATH . DS . 'views' . DS . 'common' . DS . 'header.php');
	        }
	 	    if(file_exists(APP_PATH. DS . 'views' . DS . $this->_view . DS . $this->_action . '.php')) {
	        	include (APP_PATH. DS . 'views' . DS . $this->_view . DS . $this->_action . '.php');
	 	    } else {
			    BaseController::page404();
	 	    }
	        if(file_exists(APP_PATH . DS . 'views' . DS . 'common' . DS . 'footer.php')) {
	        	include (APP_PATH . DS . 'views' . DS . 'common' . DS . 'footer.php');
	        }
	    }
	}

这个文件主要负责展开模板变量并包含对应的模板文件。

####Model

Model，数据模型，负责存取数据。一般的Web框架中都会有对象关系映射（ORM)，即一个Model类属性对应数据库记录的一列。为了简单起见，先不考虑这个。
	
	class BaseModel extends SQLQuery
	{
	    protected $_table;

	    function __construct() {

	        $this->connect(DB_HOST,DB_USER,DB_PASSWORD,DB_NAME);
	        $this->_table = strtolower(substr(get_class($this), 0, -5));
	    }

	    function __destruct() {
	    }
	}

这个类继承自SQLQuery，它封装了数据库连接的逻辑：


	class SQLQuery {
	    protected $_db;
	    protected $_result;

	    /** Connects to database **/

	    function connect($address, $account, $pwd, $name) {
	        $this->_db = @mysql_connect($address, $account, $pwd);
	        if ($this->_db != 0) {
	            if (mysql_select_db($name, $this->_db)) {
	                return 1;
	            }
	            else {
	                return 0;
	            }
	        }
	        else {
	            return 0;
	        }
	    }

	    /** Disconnects from database **/

	    function disconnect() {
	        if (@mysql_close($this->_db) != 0) {
	            return 1;
	        }  else {
	            return 0;
	        }
	    }
	    
	    function selectAll() {
	    	$query = 'select * from `'.$this->_table.'`';
	    	return $this->query($query);
	    }

		function query($query, $singleResult = 0) {

			$this->_result = mysql_query($query, $this->_db);
			$result = array();
			$field = array();
			$tempResults = array();
			$numOfFields = mysql_num_fields($this->_result);
			for ($i = 0; $i < $numOfFields; ++$i) {
			    array_push($field,mysql_field_name($this->_result, $i));
			}

			
	        while ($row = mysql_fetch_row($this->_result)) {
	            for ($i = 0;$i < $numOfFields; ++$i) {
	                $tempResults[$field[$i]] = $row[$i];
	            }
	            if ($singleResult == 1) {
	                mysql_free_result($this->_result);
	                return $tempResults;
	            }
	            array_push($result,$tempResults);
	        }
	        mysql_free_result($this->_result);
	        return($result);
		}
	    /** 释放数据库连接 **/
	    function freeResult() {
	        mysql_free_result($this->_result);
	    }
	    /** 返回错误字符串 **/
	    function getError() {
	        return mysql_error($this->_db);
	    }
	}

####实例
有了上面的mvc，就可以开始做简单的开发了：
	
	class IndexController extends BaseController
	{
		public function actDefault($defaultArg = null)
	    {
	        $this->set('defaultArg', $defaultArg);
	        $message = new MessageModel();
	        $msgs = $message->selectAll();
	        $this->set('messages', $msgs);
		}
	}

	echo is_null($defaultArg) ? 'hello world' : 'hello ' . $defaultArg;
	
    echo '<h3>留言板</h3>
	<table border=0>
	<tr><th>内容</th><th>时间</th>';
	foreach($messages as $msg)
	{
		echo '<tr>';
		echo '<td>', $msg['msg'], '</td><td>', date('Y-m-d H:i:s', $msg['addtime']), '</td>';
		echo '</tr>';
	}
	echo '</table>';

	class MessageModel extends BaseModel
	{
		
	}

在浏览器中访问 www.youly.com,如下

![example](/assets/images/mvc-example.png)

###参考
***
1、<http://anantgarg.com/2009/03/13/write-your-own-php-mvc-framework-part-1/>

2、[Apache rewrite rules](http://www.yourhtmlsource.com/sitemanagement/urlrewriting.html)
