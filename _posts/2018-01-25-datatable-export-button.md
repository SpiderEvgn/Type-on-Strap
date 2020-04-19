---
layout: post
title: 在 Laravel 上实现 DataTable 的 Excel 导出按钮
date: 2018-01-25
tags: laravel
---

> DataTable 是非常著名的 jquery 表格插件，功能强大，最近遇到一个需求是如何基于 DataTable 将数据导出至 Excel 文件（还支持 csv、pdf 等其他格式，这里我只介绍 Excel）。

最新的 laravel-datatables-buttons 插件版本是 3.0，依赖的 laravel-datatables-oracle 的版本是 8.0，而 8.0 依赖最新的 Laravel 5.5，升级框架版本风险毕竟大一点，所以暂时不想动，等到以后空了再升。所以在此版本限制的前提下，我选择了低一个版本的 DataTable 7.0，对应的是 Buttons 2.0 版本，具体步骤如下：

### 安装（升级）依赖包

1. 不管是新装还是升级，都可以直接跑这条命令：

    ```
    $ composer require yajra/laravel-datatables-oracle:^7.0
    ```
这里遇到一个很奇怪的问题，只 require 一个包的时候，composer.lock 会自动改了很多其他包的 content-hash 和 dist.url，如下图，这里先记一下，以后找到原因了再补充:
![composerLock_changed](/assets/img/posts/2018/datatable-export/composerLock_changed.png "composerLock_changed")

2. 在 config/app.php 添加声明，升级的话如果已经加过就不用了：

    ```
    'providers' => [
        // ...
        Yajra\Datatables\DatatablesServiceProvider::class,
    ],
    ```

3. 生成配置文件。这一步升级的话也需要，因为 7.0 版本的配置文件改了，而且升级必须加上 --force 的参数，否则不会覆盖原有的配置文件：
    
    ```
    $ php artisan vendor:publish --tag=datatables (--force)
    ```

4. 然后安装插件，Buttons 插件依赖 html，会自动安装：

    ```
    $ composer require yajra/laravel-datatables-buttons:^2.0
    ```

5. 在 config/app.php 添加 Buttons 的声明：

    ```
    'providers' => [
        // ...
        Yajra\Datatables\DatatablesServiceProvider::class,
        Yajra\Datatables\ButtonsServiceProvider::class,
    ],
    ```

6. 生成 Buttons 插件的配置文件：

    ```
    $ php artisan vendor:publish --tag=datatables-buttons
    
    # 根据项目改配置 config/datatables-buttons.php：
    'namespace' => [
        'base'  => 'DataTables',
        'model' => '',
    ],
    ```

### 环境配置完成，接下来开始一步步写代码

1. 对需要利用 DataTable 的模型跑命令：

    ```
    # php artisan datatables:make Model           // Model 换成你实际的模型名称
    // 输出: app/DataTables/ModelDataTable.php
    ```

2. 完成后台生成 DataTable 的代码，这里我只是举一些例子，你可以照着我的语法按实际情况修改：

    ```
    "app/DataTables/ModelDataTable.php"

    // 修改资源声明
    use App\Models\AdmissionEnquiry;

    public function dataTable()
    {
        return $this->datatables
            ->of($this->query())
            ->rawColumns(['actions'])
            ->editColumn('title', '{ { $title . " " . $first_name . " " . $last_name }}')      // Liquid 语法不允许连续两个左花括号“{ {”，所以我在其中加了个空格，真实的代码中是连在一起的
            ->editColumn('actions', '{!! \'<a href="\'.route(\'model.details\', ["id" => $id]).\'" class="btn btn-primary btn-sm"><i class="fa fa-eye"></i></a>\' !!}');
    }

    public function query()
    {
        $query = Enquiry::all();

        return $this->applyScopes($query);
    }

    public function html()
    {
        return $this->builder()
                    ->columns($this->getColumns())
                    ->minifiedAjax('')
                    ->parameters([
                        'dom'     => '<"row"<"col-sm-2"l><"col-sm-1"B><"col-sm-9"f>>rtip',
                        'buttons' => [
                            [
                                'extend'    => 'excel', 
                                'text'      => 'Export Excel', 
                                'className' => 'btn btn-success',
                            ]
                        ],
                    ]);
    }

    protected function getColumns()
    {
        return [
            ['data' => 'created_at', 'name' => 'created_at', 'title' => 'Date',  'searchable' => false],
            ['data' => 'title',      'name' => 'title',      'title' => 'Name'],
            'email',
            ['data' => 'phone',      'name' => 'phone',      'title' => 'Phone', 'orderable'  => false],
            ['data' => 'school_id',  'name' => 'school_id',  'title' => 'School'],
            'message',
            ['data' => 'actions',    'name' => 'actions',    'title' => '<i class="fa fa-cogs" aria-hidden="true"></i>', 'searchable' => false, 'orderable' => false]
        ];
    }
    ```

3. 添加路由，原本有的就不用加了：

    ```
    "routes/web.php"

    Route::get('/url', 'ModelController@index')->name('model.index');
    ```

4. 给这个路由完成页面的渲染功能：

    ```
    "app/Http/Controllers/ModelController.php"

    public function index(ModelDataTable $dataTable)
    {
        return $dataTable->render('model.index');
    }
    ```

5. 最后来到视图，新的写法和原来有很大不同：

    ```
    "resources/views/model/index.blade.php"

    @extends('layout')

    @section('content')

        {!! $dataTable->table(['class' => 'table table-bordered table-striped table-hover']) !!}

    @endsection

    @push('scripts')

        <link rel="stylesheet" href="/vendor/datatables/buttons.dataTables.min.css">    // 这两个样式文件
        <script src="/vendor/datatables/dataTables.buttons.min.js"></script>
        <script src="/vendor/datatables/buttons.server-side.js"></script>
        {!! $dataTable->scripts() !!}
        <script>
            var $excelButton = $('#dataTableBuilder_wrapper').find('.dt-buttons').find('a');
            $excelButton.removeClass('dt-button');
        </script>

    @endpush
    ```

6. 至此，整个利用 laravel-datatables-buttons 插件的 excel 导出功能就做好了。想要快速应用的同学看到这里就够了，模仿着我的代码稍微改动就可以直接用了。

### 下面我要讲讲一些细节和遇到的问题，内容比较多，如果你想要继续深入了解或做一些定制，就继续读下去。

这个新的结构和之前版本的写法有很大不同，原来的框架类似如下结构：

    ```
    "resources/views/model/index.blade.php"

    @extends('layout')

    @section('content')

        <div class="table-responsive">
            <table class="table table-bordered table-striped table-hover" id="models-table">
                <thead>
                <tr>
                    <th>Date</th>
                    <th>Title</th>
                    <th><i class="fa fa-cogs" aria-hidden="true"></i></th>
                </tr>
                </thead>
            </table>
        </div>

    @endsection

    @push('scripts')
        <script>
            $(function() {
                $('#models-table').DataTable({
                    processing: true,
                    serverSide: true,
                    ajax: {
                        url: '{!! route('model.data') !!}',
                        type: 'POST'
                    },
                    order: [ [0, 'desc'] ],
                    columns: [
                        { data: 'created_at', name: 'created_at' },
                        { data: 'title', name: 'title' },
                        { data: 'actions', name: 'actions', searchable: false, sortable: false }
                    ]
                });
            });
        </script>
    @endpush
    ```

视图层需要手动画出 table 轮廓，然后利用 DataTable 的 ajax 请求数据。如此，在后端就需要有两个 action，第一个 index 用来渲染页面，第二个就是 ajax 的 route('model.data') 方法用来生成 DataTable 数据。

而 laravel-datatables-buttons 通过 datatable:make 新建了一个继承自 DataTable 的类，利用这个类在后端搭建起了整个 dataTable 的框架，而不再是前端用 script 做，所以很多具体的写法要有调整。

### 视图 view 的变化

前端的框架统统交给 DataTable 直接完成了，不用再手动画 table，并且 table 可以很方便的传入 class 参数定义：

```
{!! $dataTable->table(['class' => 'table table-bordered table-striped table-hover']) !!}
```

js 层也不用手动调用 ajax，直接调用 dataTable 的 scripts() 方法：

```
<link rel="stylesheet" href="/vendor/datatables/buttons.dataTables.min.css">
<script src="/vendor/datatables/dataTables.buttons.min.js"></script>
<script src="/vendor/datatables/buttons.server-side.js"></script>

{!! $dataTable->scripts() !!}
```

但是需要加上三个样式文件，我把它们下载到本地了，[官网](https://yajrabox.com/docs/laravel-datatables/7.0/buttons-starter)有链接，其中 buttons.server-side.js 是随安装包一起的。

此外，你会发现我还写了如下 js 代码：

```
<script>
    var $excelButton = $('#dataTableBuilder_wrapper').find('.dt-buttons').find('a');
    $excelButton.removeClass('dt-button');
</script>
```
这个我最后会讲到再解释。

### 如何改变 column 名字和内容？

getColumns() 方法最简单的用法是只用一个一维数组列出所有 Model 模型的字段，就像官网的教程：

```
"app/DataTables/ModelDataTable.php"

protected function getColumns()
{
    return [
        'id',
        'name',
        'email',
        'created_at',
        'updated_at',
    ];
}
```

这样的话，前端表格默认就会以字段名（首字母大写）为列名，字段值为内容填充表格。可是，很多情况下我们需要改变列名，比如 create_at 字段我们需要在网页上显示的表格列名是 Date。还有，我们希望自定义列是否支持搜索或者排序。做法如下：

```
['data' => 'created_at', 'name' => 'created_at', 'title' => 'Date', 'searchable' => false, 'orderable' => false]
```

注意，这里排序是 orderable，原来是 sortable，很坑。。。

这里自定义了列名，那么如何修改值呢？我们要回到最上面的 dataTable() 方法，写法和原来很相似，

```
"app/DataTables/ModelDataTable.php"

public function dataTable()
{
    return $this->datatables
        ->of($this->query())
        ->rawColumns(['actions'])
        ->editColumn('title', '{ { $title . " " . $first_name . " " . $last_name }}')
        ->editColumn('actions', '{!! \'<a href="\'.route(\'model.details\', ["id" => $id]).\'" class="btn btn-primary btn-sm"><i class="fa fa-eye"></i></a>\' !!}');
}
```

两个点注意：
>1. 像如上自定义了 actions 的 html 内容，必须要用 rawColumns 声明；
>2. 变量一定是 query() 方法查询到的字段，这个字段不一定是 Model 原来的字段，可是是用 sql select AS 定义的。

### 接下来深入讲讲最重要的环节: button 的定制

官网的示例如下（因为我只用 excel 所以 buttons 数组只留了一个 'excel' ）：

```
->parameters([
    'dom'          => 'Bfrtip',
    'buttons'      => ['export', 'print', 'reset', 'reload'],
])
```

如果你这么写的话会发现 table 的顶端变成了这样：
![button_without_paging](/assets/img/posts/2018/datatable-export/button_without_paging.png "button_without_paging")

而原来是这样的：
![paging_without_button](/assets/img/posts/2018/datatable-export/paging_without_button.png "paging_without_button")

新的 button 按钮把原来的表格长度选择控件覆盖掉了。这其实是 dom 这个参数捣的鬼，之所以说是捣鬼，就是因为它实在是恶心，'Bfrtip'，谁一眼能知道这个诡异的字符串是什么意思？甚至你看了文档([DOM positioning](https://datatables.net/examples/basic_init/dom.html)) 也要花不少时间测试理解。

查了文档发现 dom 就是用来布局 table 周围的一些组件的。'B' 就代表 Button，而缺少的 'l' 就是消失的组件：length changing。'Bfrtip' 字符串的排列顺序决定了组件的位置，非常恶心吧！更恶心的是，你以为把 'l' 加上去，变成 'lBfrtip' 就好了吗？长度选择控件就和按钮一起并排出现在左上方吗？并不是！这两个控件是上下交错的！所以最终你要并排美观的显示，还要更深入地定制化手写 div，我简单用了 bootstrap 的 grid，最后的结果就成了这样：

```
'dom' => '<"row"<"col-sm-1"l><"col-sm-1"B><"col-sm-10"f>>rtip'
```
怎么样？恶心至极！

button 自定义样式的恶心还没完，这只是刚刚开始！排好了位置，然后如果你想修改 button 的样式，你会遇到一个更大的坑。

首先查看[buttons 文档](https://datatables.net/reference/option/#buttons)，比如 button 有 text 这个 option，可以自定义 button 名称，然后还发现有个 className 的 option，名字看起来非常诱人，这个应该就是自定义 class 的参数了吧，于是我更改如下：

```
'buttons' => [
    [
        'extend'    => 'excel', 
        'text'      => 'Export Excel', 
        'className' => 'btn btn-success',
    ]
]
```

结果却是：
![button_without_class](/assets/img/posts/2018/datatable-export/button_without_class.png "button_without_class")

打开 debug 面板，发现 class 是 "dt-button buttons-excel btn btn-success"。
这是什么鬼？我不是把整个 className 自定义了更改了吗？

于是乎找到了这篇关于 className 的[详细文档](https://datatables.net/reference/option/buttons.dom)，原来默认的 dt-buttons 这个 class 是不会被覆盖的？！

WTF。。。逗我呢？还能不能更恶心一点！

找了各种方法无法重写 class，于是最后只能用了一个比它更恶心的方法，就是我一开始提到的，用一段 jquery 手动 removeClass。真是魔高一尺，道高一丈，就比谁更恶心！

```
<script>
    var $excelButton = $('#dataTableBuilder_wrapper').find('.dt-buttons').find('a');
    $excelButton.removeClass('dt-button');
</script>
```

---

## 参考资料：

* [Laravel DataTable 官网](https://yajrabox.com/docs/laravel-datatables)
* [DataTable buttons 中文网](http://www.datatables.club/extensions/buttons/)
* [https://datatables.net/](https://datatables.net/)




