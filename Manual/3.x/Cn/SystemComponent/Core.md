# Core

> 核心类: EasySwoole\EasySwoole\Core

Core 它是一个单例类(use EasySwoole\Component\Singleton)
```php
<?php
/**
 * Created by PhpStorm.
 * User: yf
 * Date: 2018/5/28
 * Time: 下午6:07
 */

namespace EasySwoole\EasySwoole;


use EasySwoole\Actor\Actor;
use EasySwoole\Component\Context\ContextManager;
use EasySwoole\Component\Di;
use EasySwoole\Component\Singleton;
use EasySwoole\EasySwoole\AbstractInterface\Event;
use EasySwoole\EasySwoole\Console\TcpService;
use EasySwoole\EasySwoole\Crontab\Crontab;
use EasySwoole\EasySwoole\Swoole\EventHelper;
use EasySwoole\EasySwoole\Swoole\EventRegister;
use EasySwoole\EasySwoole\Swoole\Task\QuickTaskInterface;
use EasySwoole\FastCache\Cache;
use EasySwoole\Http\Dispatcher;
use EasySwoole\Http\Message\Status;
use EasySwoole\Http\Request;
use EasySwoole\Http\Response;
use EasySwoole\Trace\Bean\Location;
use EasySwoole\EasySwoole\Swoole\PipeMessage\Message;
use EasySwoole\EasySwoole\Swoole\PipeMessage\OnCommand;
use EasySwoole\EasySwoole\Swoole\Task\AbstractAsyncTask;
use EasySwoole\EasySwoole\Swoole\Task\SuperClosure;
use Swoole\Server\Task;

////////////////////////////////////////////////////////////////////
//                          _ooOoo_                               //
//                         o8888888o                              //
//                         88" . "88                              //
//                         (| ^_^ |)                              //
//                         O\  =  /O                              //
//                      ____/`---'\____                           //
//                    .'  \\|     |//  `.                         //
//                   /  \\|||  :  |||//  \                        //
//                  /  _||||| -:- |||||-  \                       //
//                  |   | \\\  -  /// |   |                       //
//                  | \_|  ''\---/''  |   |                       //
//                  \  .-\__  `-`  ___/-. /                       //
//                ___`. .'  /--.--\  `. . ___                     //
//            \  \ `-.   \_ __\ /__ _/   .-` /  /                 //
//      ========`-.____`-.___\_____/___.-`____.-'========         //
//                           `=---='                              //
//      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^        //
//         佛祖保佑       永无BUG       永不修改                     //
////////////////////////////////////////////////////////////////////


class Core
{
    use Singleton;

    private $isDev = true;

    function __construct()
    {
        defined('SWOOLE_VERSION') or define('SWOOLE_VERSION',intval(phpversion('swoole')));
        defined('EASYSWOOLE_ROOT') or define('EASYSWOOLE_ROOT',realpath(getcwd()));
        defined('EASYSWOOLE_SERVER') or define('EASYSWOOLE_SERVER',1);
        defined('EASYSWOOLE_WEB_SERVER') or define('EASYSWOOLE_WEB_SERVER',2);
        defined('EASYSWOOLE_WEB_SOCKET_SERVER') or define('EASYSWOOLE_WEB_SOCKET_SERVER',3);
        //定义swoole easyswoole版本常量和框架启动的服务类型常量
    }

    /**
     * 设置框架是否与开发/生产环境启动
     * setIsDev
     * @param bool $isDev
     * @return $this
     * @author Tioncico
     * Time: 15:27
     */
    function setIsDev(bool $isDev)
    {
        $this->isDev = $isDev;
        //变更这里的时候，例如在全局的事件里面修改的，，重新加载配置项
        $this->loadEnv();
        return $this;
    }

    /**
     * 返回是否以开发环境启动
     * isDev
     * @return bool
     * @author Tioncico
     * Time: 15:27
     */
    function isDev():bool
    {
        return $this->isDev;
    }

    /**
     * 框架初始化事件
     * initialize
     * @return $this
     * @author Tioncico
     * Time: 15:28
     */
    function initialize()
    {
        //检查全局事件文件是否存在.
        $file = EASYSWOOLE_ROOT . '/EasySwooleEvent.php';
        if(file_exists($file)){
            require_once $file;
            try{
                $ref = new \ReflectionClass('EasySwoole\EasySwoole\EasySwooleEvent');
                if(!$ref->implementsInterface(Event::class)){
                    die('global file for EasySwooleEvent is not compatible for EasySwoole\EasySwoole\EasySwooleEvent');
                }
                unset($ref);
            }catch (\Throwable $throwable){
                die($throwable->getMessage());
            }
        }else{
            die('global event file missing');
        }
        //先加载配置文件
        $this->loadEnv();
        //执行框架初始化事件
        EasySwooleEvent::initialize();
        //临时文件和Log目录初始化
        $this->sysDirectoryInit();
        //注册错误回调
        $this->registerErrorHandler();
        return $this;
    }

    function createServer()
    {
        //获取服务配置
        $conf = Config::getInstance()->getConf('MAIN_SERVER');
        //创建swooleServer服务
        ServerManager::getInstance()->createSwooleServer(
            $conf['PORT'],$conf['SERVER_TYPE'],$conf['LISTEN_ADDRESS'],$conf['SETTING'],$conf['RUN_MODEL'],$conf['SOCK_TYPE']
        );
        //注册服务回调事件
        $this->registerDefaultCallBack(ServerManager::getInstance()->getSwooleServer(),$conf['SERVER_TYPE']);
        //注册其他事件
        EasySwooleEvent::mainServerCreate(ServerManager::getInstance()->getMainEventRegister());
        //创建主服务后，创建Tcp子服务
        (new TcpService(Config::getInstance()->getConf('CONSOLE')));
        return $this;
    }

    function start()
    {
        //给主进程也命名
        $serverName = Config::getInstance()->getConf('SERVER_NAME');
        if(PHP_OS != 'Darwin'){
            cli_set_process_title($serverName);
        }
        //注册crontab进程
        Crontab::getInstance()->__run();
        //注册fastCache进程
        if(Config::getInstance()->getConf('FAST_CACHE.PROCESS_NUM') > 0){
            Cache::getInstance()->setTempDir(EASYSWOOLE_TEMP_DIR)
                    ->setProcessNum(Config::getInstance()
                    ->getConf('FAST_CACHE.PROCESS_NUM'))
                    ->setServerName($serverName)
                    ->attachToServer(ServerManager::getInstance()->getSwooleServer());
        }

        //执行Actor注册进程
        Actor::getInstance()->setTempDir(EASYSWOOLE_TEMP_DIR)->setServerName($serverName)->attachToServer(ServerManager::getInstance()->getSwooleServer());
        //启动swoole server服务
        ServerManager::getInstance()->start();
    }

    private function sysDirectoryInit():void
    {
        //创建临时目录    请以绝对路径，不然守护模式运行会有问题
        $tempDir = Config::getInstance()->getConf('TEMP_DIR');
        if(empty($tempDir)){
            $tempDir = EASYSWOOLE_ROOT.'/Temp';
            Config::getInstance()->setConf('TEMP_DIR',$tempDir);
        }
        if(!is_dir($tempDir)){
            mkdir($tempDir);
        }
        defined('EASYSWOOLE_TEMP_DIR') or define('EASYSWOOLE_TEMP_DIR',$tempDir);

        $logDir = Config::getInstance()->getConf('LOG_DIR');
        if(empty($logDir)){
            $logDir = EASYSWOOLE_ROOT.'/Log';
            Config::getInstance()->setConf('LOG_DIR',$logDir);
        }
        if(!is_dir($logDir)){
            mkdir($logDir);
        }
        defined('EASYSWOOLE_LOG_DIR') or define('EASYSWOOLE_LOG_DIR',$logDir);

        //设置默认文件目录值
        Config::getInstance()->setConf('MAIN_SERVER.SETTING.pid_file',$tempDir.'/pid.pid');
        Config::getInstance()->setConf('MAIN_SERVER.SETTING.log_file',$logDir.'/swoole.log');
        //设置目录
        Logger::getInstance($logDir);
    }

    /**
     * 注册错误处理回调
     * registerErrorHandler
     * @throws \Throwable
     * @author Tioncico
     * Time: 15:35
     */
    private function registerErrorHandler()
    {
        ini_set("display_errors", "On");
        error_reporting(E_ALL | E_STRICT);
        //尝试获取Di的错误处理回调
        $userHandler = Di::getInstance()->get(SysConst::ERROR_HANDLER);
        if(!is_callable($userHandler)){
            $userHandler = function($errorCode, $description, $file = null, $line = null){
                $l = new Location();
                $l->setFile($file);
                $l->setLine($line);
                Trigger::getInstance()->error($description,$l);
            };
        }
        //设置错误处理回调
        set_error_handler($userHandler);

        //尝试获取di脚本终止回调函数
        $func = Di::getInstance()->get(SysConst::SHUTDOWN_FUNCTION);
        if(!is_callable($func)){
            $func = function (){
                $error = error_get_last();
                if(!empty($error)){
                    $l = new Location();
                    $l->setFile($error['file']);
                    $l->setLine($error['line']);
                    Trigger::getInstance()->error($error['message'],$l);
                }
            };
        }
        //注册脚本终止回调
        register_shutdown_function($func);
    }

    /**
     * 注册默认的服务回调事件
     * registerDefaultCallBack
     * @param \swoole_server $server
     * @param int            $serverType
     * @throws \Throwable
     * @author Tioncico
     * Time: 15:36
     */
    private function registerDefaultCallBack(\swoole_server $server,int $serverType)
    {
        //如果主服务仅仅是swoole server，那么设置默认onReceive为全局的onReceive
        if($serverType === EASYSWOOLE_SERVER){
            $socketType = Config::getInstance()->getConf('MAIN_SERVER.SOCK_TYPE');
            if(in_array($socketType,[SWOOLE_TCP,SWOOLE_TCP6])){
                ServerManager::getInstance()->getMainEventRegister()->add(EventRegister::onReceive,function (){
                    ContextManager::getInstance()->destroy();
                });
            }else if(in_array($socketType,[SWOOLE_UDP,SWOOLE_UDP6])){
                ServerManager::getInstance()->getMainEventRegister()->add(EventRegister::onPacket,function (){
                    ContextManager::getInstance()->destroy();
                });
            }
        }else{
            //http回调
            $namespace = Di::getInstance()->get(SysConst::HTTP_CONTROLLER_NAMESPACE);
            if(empty($namespace)){
                $namespace = 'App\\HttpController\\';
            }
            $depth = intval(Di::getInstance()->get(SysConst::HTTP_CONTROLLER_MAX_DEPTH));
            $depth = $depth > 5 ? $depth : 5;
            $max = intval(Di::getInstance()->get(SysConst::HTTP_CONTROLLER_POOL_MAX_NUM));
            if($max == 0){
                $max = 15;
            }
            $waitTime = intval(Di::getInstance()->get(SysConst::HTTP_CONTROLLER_POOL_WAIT_TIME));
            if($waitTime == 0){
                $waitTime = 5;
            }
            $dispatcher = new Dispatcher($namespace,$depth,$max);
            $dispatcher->setControllerPoolWaitTime($waitTime);
            $httpExceptionHandler = Di::getInstance()->get(SysConst::HTTP_EXCEPTION_HANDLER);
            if(!is_callable($httpExceptionHandler)){
                $httpExceptionHandler = function ($throwable,$request,$response){
                    $response->withStatus(Status::CODE_INTERNAL_SERVER_ERROR);
                    $response->write(nl2br($throwable->getMessage()."\n".$throwable->getTraceAsString()));
                    Trigger::getInstance()->throwable($throwable);
                };
                Di::getInstance()->set(SysConst::HTTP_EXCEPTION_HANDLER,$httpExceptionHandler);
            }
            $dispatcher->setHttpExceptionHandler($httpExceptionHandler);

            EventHelper::on($server,EventRegister::onRequest,function (\swoole_http_request $request,\swoole_http_response $response)use($dispatcher){
                $request_psr = new Request($request);
                $response_psr = new Response($response);
                try{
                    if(EasySwooleEvent::onRequest($request_psr,$response_psr)){
                        $dispatcher->dispatch($request_psr,$response_psr);
                    }
                }catch (\Throwable $throwable){
                    call_user_func(Di::getInstance()->get(SysConst::HTTP_EXCEPTION_HANDLER),$throwable,$request_psr,$response_psr);
                }finally{
                    try{
                        EasySwooleEvent::afterRequest($request_psr,$response_psr);
                    }catch (\Throwable $throwable){
                        call_user_func(Di::getInstance()->get(SysConst::HTTP_EXCEPTION_HANDLER),$throwable,$request_psr,$response_psr);
                    }
                }
                $response_psr->__response();
                ContextManager::getInstance()->destroy();
            });

            if($serverType == EASYSWOOLE_WEB_SOCKET_SERVER){
                ServerManager::getInstance()->getMainEventRegister()->add(EventRegister::onMessage,function (){
                    ContextManager::getInstance()->destroy();
                });
            }
        }
        //注册默认的on task,finish  不经过 event register。因为on task需要返回值。不建议重写onTask,否则es自带的异步任务事件失效
        //其次finish逻辑在同进程中实现、
        if(Config::getInstance()->getConf('MAIN_SERVER.SETTING.task_enable_coroutine')){
            EventHelper::on($server,EventRegister::onTask,function (\swoole_server $server, Task $task){
                $taskObj = $task->data;
                if(is_string($taskObj) && class_exists($taskObj)){
                    $ref = new \ReflectionClass($taskObj);
                    if($ref->implementsInterface(QuickTaskInterface::class)){
                        try{
                            $taskObj::run($server,$task->id,$task->worker_id,$task->flags);
                        }catch (\Throwable $throwable){
                            Trigger::getInstance()->throwable($throwable);
                        }
                        return;
                    }else if($ref->isSubclassOf(AbstractAsyncTask::class)){
                        $taskObj = new $taskObj;
                    }
                }
                if($taskObj instanceof AbstractAsyncTask){
                    try{
                        $ret = $taskObj->__onTaskHook($task->id,$task->worker_id,$task->flags);
                        if($ret !== null){
                            $taskObj->__onFinishHook($ret,$task->id);
                        }
                    }catch (\Throwable $throwable){
                        Trigger::getInstance()->throwable($throwable);
                    }
                }else if($taskObj instanceof SuperClosure){
                    try{
                        return $taskObj( $server, $task->id,$task->worker_id,$task->flags);
                    }catch (\Throwable $throwable){
                        Trigger::getInstance()->throwable($throwable);
                    }
                }else if(is_callable($taskObj)){
                    try{
                        call_user_func($taskObj,$server,$task->id,$task->worker_id,$task->flags);
                    }catch (\Throwable $throwable){
                        Trigger::getInstance()->throwable($throwable);
                    }
                }
                return null;
            });
        }else{
            EventHelper::on($server,EventRegister::onTask,function (\swoole_server $server, $taskId, $fromWorkerId,$taskObj){
                if(is_string($taskObj) && class_exists($taskObj)){
                    $ref = new \ReflectionClass($taskObj);
                    if($ref->implementsInterface(QuickTaskInterface::class)){
                        try{
                            $taskObj::run($server,$taskId,$fromWorkerId);
                        }catch (\Throwable $throwable){
                            Trigger::getInstance()->throwable($throwable);
                        }
                        return;
                    }else if($ref->isSubclassOf(AbstractAsyncTask::class)){
                        $taskObj = new $taskObj;
                    }
                }
                if($taskObj instanceof AbstractAsyncTask){
                    try{
                        $ret = $taskObj->__onTaskHook($taskId,$fromWorkerId);
                        if($ret !== null){
                            $taskObj->__onFinishHook($ret,$taskId);
                        }
                    }catch (\Throwable $throwable){
                        Trigger::getInstance()->throwable($throwable);
                    }
                }else if($taskObj instanceof SuperClosure){
                    try{
                        return $taskObj( $server, $taskId, $fromWorkerId);
                    }catch (\Throwable $throwable){
                        Trigger::getInstance()->throwable($throwable);
                    }
                }else if(is_callable($taskObj)){
                    try{
                        call_user_func($taskObj,$server,$taskId,$fromWorkerId);
                    }catch (\Throwable $throwable){
                        Trigger::getInstance()->throwable($throwable);
                    }
                }
                return null;
            });
        }
        EventHelper::on($server,EventRegister::onFinish,function (){
            //空逻辑
        });

        //通过pipe通讯，也就是processAsync投递的闭包任务，是没有taskId信息的，因此参数传递默认-1
        OnCommand::getInstance()->set('TASK',function (\swoole_server $server,$taskObj,$fromWorkerId){
            //闭包任务无法再次二次序列化,因此直接执行
            if($taskObj instanceof SuperClosure){
                try{
                    call_user_func($taskObj,$server,-1,$fromWorkerId);
                }catch (\Throwable $throwable){
                    Trigger::getInstance()->throwable($throwable);
                }
            }else{
                $server->task($taskObj);
            }
        });

        EventHelper::on($server,EventRegister::onPipeMessage,function (\swoole_server $server,$fromWorkerId,$data){
            $message = unserialize($data);
            if($message instanceof Message){
                OnCommand::getInstance()->hook($message->getCommand(),$server,$message->getData(),$fromWorkerId);
            }else{
                Trigger::getInstance()->error("data :{$data} not packet as an Message Instance");
            }
        });

        //注册默认的worker start
        EventHelper::registerWithAdd(ServerManager::getInstance()->getMainEventRegister(),EventRegister::onWorkerStart,function (\swoole_server $server,$workerId){
            if(PHP_OS != 'Darwin'){
                $name = Config::getInstance()->getConf('SERVER_NAME');
                if( ($workerId < Config::getInstance()->getConf('MAIN_SERVER.SETTING.worker_num')) && $workerId >= 0){
                    $type = 'Worker';
                }else{
                    $type = 'TaskWorker';
                }
                cli_set_process_title("{$name}.{$type}.{$workerId}");
            }
        });
    }

    /**
     * 加载配置文件
     * loadEnv
     * @throws \Exception
     * @author Tioncico
     * Time: 15:48
     */
    private function loadEnv()
    {
        //加载之前，先清空原来的
        //判断dev环境
        if($this->isDev){
            $file  = EASYSWOOLE_ROOT.'/dev.php';
        }else{
            $file  = EASYSWOOLE_ROOT.'/produce.php';
        }
        Config::getInstance()->loadEnv($file);
    }
}
```