## 相关文档
-  openshift:[https://docs.openshift.org/latest/rest_api/openshift_v1.html](https://docs.openshift.org/latest/rest_api/openshift_v1.html)
- kubernetes: [https://docs.openshift.org/latest/rest_api/kubernetes_v1.html](https://docs.openshift.org/latest/rest_api/kubernetes_v1.html)
- datafoundry: [http://asiainfoldp.github.io/datafactory/](http://asiainfoldp.github.io/datafactory/)
- angular: [http://docs.angularjs.cn/api/ngResource/service/$resource](http://docs.angularjs.cn/api/ngResource/service/$resource)

## 开始

### 环境
- git
- nginx
- npm工具包（nodejs）
- bower工具包（安装命令: npm install bower -g）

### 准备工作（默认环境需要工具均已安装）

1. 克隆代码。git clone https://github.com/asiainfoLDP/datafoundry_web.git
> 认证需要依赖另外一个项目，git clone https://github.com/asiainfoLDP/datafoundry_proxy.git

2. 安装依赖。进入datafoundry_web目录下，执行：
   - npm install
   - bower install
   
3. 配置环境变量，DATAFOUNDRY_APISERVER_ADDR=openshift服务器地址
4. 配置nginx。复制nginx.conf中的内容到本地，修改配置符合本地项目路径，检查无误后启动nginx
5. 启动认证服务。进入datafoundry_proxy目录下，执行./oauth-proxy
6. 浏览器中访问。

### 代码打包配置（app/main.js|conf/build.js)

- 代码通过require.js组织，实现资源文件按需加载
- 所有第三方依赖需要在bower.json和package.json中设置，具体参考已有配置
- 新引入第三方依赖，需要在app/main.js中进行配置
  1. 在app/main.js中指定模块路径，require.config({paths:...})，具体参考已有配置。
  2. 在app/main.js中指定依赖关系，require.config({shim:...})，具体参考已有配置。
- 代码通过r.js打包，打包后代码放置在dist目录下
- 新引入第三方依赖，需要在conf/build.js中进行配置
  1. 在app/main.js中指定模块路径，{paths:...}，具体参考已有配置。
  2. 在app/main.js中指定依赖关系，{shim:...}，具体参考已有配置。

### 打包
> 执行：./release.sh  

## 开发

### 资源resource（app/resource.js）

> GET/POST/PUT/DELETE /oapi/v1/namespaces/{namespace}/builds/{name}

resource定义

```js
.factory('Build', ['$resource', 'GLOBAL', function($resource, GLOBAL){
    //构建列表/详情/创建/删除
    var Build = $resource(GLOBAL.host + '/namespaces/:namespace/builds/:name', {name: '@name', namespace: '@namespace'}, {
        create: { method: 'POST'},
        put: { method: 'PUT'}
    });
    //构建日志查询
    Build.log = $resource(GLOBAL.host + '/namespaces/:namespace/builds/:name/log', {name: '@name', namespace: '@namespace'})
    return Build;
}])
```

调用方式
```js
.controller('BuildCtrl', ['$scope', 'Build', function($scope, Build){
    //调用方式1
    Build.get(function(res){

    }, function(res){
        //错误处理代码
    });
    //调用方式2
    Build.get({namespace: 'foo'}, function(res){
        $scope.data = res;  //res为接口返回的数据，一个json对象。
    }, function(res){
        //错误处理代码
    });
    //调用方式3
    Build.create({namespace: 'foo', name: 'bar'}, data, function(res){
        $scope.data = res;
    }, function(res){
        //错误处理代码
    });
}])
```

- 通过 .factory('Name', [function(){}]) 定义
- 可以通过依赖注入的方式使用系统或自定义的服务，例如：$resource, GLOBAL等
- 方法返回一个对象，对象内通过$resource定义与接口的交互
- $resource(url, param, options), 参考：[http://docs.angularjs.cn/api/ngResource/service/$resource](http://docs.angularjs.cn/api/ngResource/service/$resource)

### 路由router（app/router.js）
```js
.state('console', {
    url: '/console',  //浏览器显示的地址
    templateUrl: 'views/console/console.html',  //模板，页面显示
    controller: 'ConsoleCtrl',  //控制器，业务处理
    resolve: {
        /*这部分实现按需加载，固定写法*/
        dep: ['$ocLazyLoad', function ($ocLazyLoad) {
            return $ocLazyLoad.load('views/console/console.js')
        }],
        /*这个在加载时会阻塞程序，直到拿到数据才会继续运行*/
        user: ['$rootScope', 'User', function($rootScope, User){
            if($rootScope.user){
                return $rootScope.user;
            }
            return User.get({name: '~'}).$promise;
        }]
    },
    abstract: true
})
.state('console.build', {
    url: '/build',
    templateUrl: 'views/build/build.html',
    controller: 'BuildCtrl',
    resolve: {
        dep: ['$ocLazyLoad', function ($ocLazyLoad) {
            return $ocLazyLoad.load('views/build/build.js')
        }]
    }
})
```

- 路由中url可以指定"/:name"这种形式，通过$stateParams.name获取该参数，例如：url:/build/:name则访问/build/foo，$stateParams.name == 'foo'
- 具体参考[http://angular-ui.github.io/ui-router/site/#/api/ui.router.state.$state](http://angular-ui.github.io/ui-router/site/#/api/ui.router.state.$state)

### 模块(app/components)
- 模块定义成指令形式

```js
angular.module("console.card", [
    {
        files: ['components/card/card.css']   //按需加载资源
    }
])
.directive('fooBar', [function () {
    return {
        restrict: 'EA',
        replace: true,
        templateUrl: 'components/foobar/foobar.html',
        scope: {
          foo: '=',
          bar: '@',
          foobar: '&'
        }
        controller: ['$scope', function($scope){
          ...
        }]
    };
}]);
```
html
```html
<div class="foo-bar">
    ...
</div>
```
调用
- 在js中引入该模块的js文件
- 在html页面中通过“<foo-bar></foo-bar>”的方式调用

### 页面（app/views）
```js
angular.module('console.build', [
    {
        files: [
            'components/searchbar/searchbar.js',  //依赖的模块
            'views/build/build.css'               //样式文件
        ]
    }
])
.controller('BuildCtrl', ['$scope', function ($scope) {
    ...
}]);
```
说明
- 每个路由对应一个view，一个view包含一个controller，一个template
- 控制器中放置业务代码，可以引用其他模块，服务。
- 模板中主要做布局，然后由模块填充内容

## 框架介绍

### 常用指令
- ng-repeat
- ng-if/ng-show
- ng-model
- ng-bind/{{}}
- ng-class/ng-style
- ng-include
- ...
> 参考 angular文档

### controller

- 控制器存放业务逻辑代码
- 需要在页面上展示的内容通过$scope绑定到页面，例如：$scope.data = data;页面上通过{{data}}进行使用

js
```js
.controller('BuildCtrl', ['$scope', 'Build', function($scope, Build){
    Build.get({namespace: 'foo'}, function(res){
        $scope.data = res;
    }, function(res){
        //错误处理代码
    });
}])
```

html
```html
<div ng-controller="BuildCtrl">
  <p>{{data.name}}</p>
  <input type="text" ng-model="data.name">
</div>
```

### service

- service类似于全局方法，可以在其他模块内通过依赖注入的方式使用
- 常量(constant)、变量(value)、工厂函数(factory)、服务(service)等都是服务

```js
.constant('GLOBAL', {
  size: 10
})
.factory('FooService', ['$scope', function ($scope) {
  return {
      foo: function (param) {
          //方法代码
      }
  };
}])
.service('Confirm', ['$scope', function ($scope) {
  this.func = function (title, txt) {
      //方法代码
  };
}]);
```

例
```js
.service('Confirm', ['$uibModal', function ($uibModal) {
  this.open = function (title, txt) {
      return $uibModal.open({
          templateUrl: 'pub/tpl/confirm.html',
          size: 'default',
          controller: ['$scope', '$uibModalInstance', function ($scope, $uibModalInstance) {
              $scope.title = title;
              $scope.txt = txt;
              $scope.ok = function () {
                  $uibModalInstance.close(true);
              };
              $scope.cancel = function () {
                  $uibModalInstance.dismiss();
              };
          }]
      }).result;
  };
}])
```

### directive
```js
.directive('fullHeight', [function() {
    return function(scope, element, attr) {
        var height = document.documentElement.clientHeight - 70 + 'px';
        element.css({
            'min-height': height
        });
    };
}])
.directive('fooDirective', [function () {
    return {
        restrict: 'EA',
        replace: true,
        scope: {
          foo: '=',
          bar: '@',
          foobar: '&'
        }
        templateUrl: 'components/sidebar/sidebar.html',
        link: function(scope){

        },
        controller: ['$state', '$scope', function($state, $scope){
            $scope.$state = $state;
        }]
    }
}])
```

### filter
```js
.filter('fooFilter',[function(){
    return function(foo, bar){
        return 'prex' + foo + bar;
    };
}])
```
html
```html
<div>
  {{sth | fooFilter : 'bar'}}
</div>
```
