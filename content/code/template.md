---
title: "代码生成大杀器-模板引擎"
date: 2018-11-26T20:50:36+08:00
draft: false
tags: ['模板引擎', 'python', '代码生成']
---

作为一个爱偷懒的程序员，对代码生成格外的钟爱。一般的情况下使用简单的python脚本：字符串拼接 + 循环就可以搞定了，但是复杂点的代码生成就搞不定了，或者比较麻烦。

代码生成大杀器有三：宏(元编程)，AST 和 模板引擎。 这里的宏，指的是lisp中的宏，lisp已经在程序员心中已经封神，相信大部分程序员都听说过“代码和数据同构”的威力。更广泛的来说，宏是元编程的一种，魔幻的Ruby对元编程支持的非常好，虽然Rails逐渐式微了，ruby也在走下坡路，但是很多人依然对Ruby元编程的威力念念不忘。java通过Annotation Processor生成代码可以归类为AST，类似的还有[JavaParser](http://javaparser.org/#generate)... 但使用最广泛，使用起来最简单直白的还是模板引擎。

因为专注于小型代码生成，所以设计上有以下特征：

1. 语法要求上比较灵活，可读性要好
2. 模板引擎的渲染性能不是很重要
3. 依赖要少，使用要简洁，不需要配置配置环境
4. 不需要支持模板宏，模板嵌套等特性

### 模板语法设计如下

```python
%{fields = jclass.fields}%

private {{jclass.name}}(Builder builder) {
    %{for field in fields:}%
    this.{{field.name}} = builder.{{field.name}};
    %{end}%
}

public static class Builder {

    %{for field in fields:}%
    private {{field.jtype}} {{field.name}} = null;
    %{end}%

    %{for field in fields:}%
    %{if field.comment:}%
    /**
     * {{field.comment}}
     */
     %{end}%
    public Builder set{{field.name.title()}}({{field.jtype}} {{field.name}}) {
        this.{{field.name}} = {{field.name}};
        return this;
    }

    %{end}%
    public {{jclass.name}} build() {
        return new {{jclass.name}}(this);
    }
}
```
1. `%{code}%` 代表任意合法的python表达式
2. `{{}}` 代表数据占位符, `{{}}` 内部的字符串会被求值，然后替换。比较类似python3 的 fstring
3. `%{end}%`代表一个表达式block结束

### 实现思路
1. 将模板改写为合法的python代码，用来渲染模板
2. 执行生成的python渲染代码
3. 渲染代码根据传入的数据，生成字符串输出

### 使用方式如下:
```python
print(Template(TMPL_BUILDER).render({'jclass': data}))
```
1. render 函数需要传递一个字典，用于替换模板中的数据占位符

### Template 实现
```python
class Template:
    def __init__(self, tmpl: str):
        self.tmpl = tmpl

    @staticmethod
    def _split_template(tmpl: str) -> Sequence[Tuple[str, str]]:

        segments = re.finditer(
            r'(?<=\n).*?%\{(?P<code>.*?)\}%.*?\n|\{\{(?P<data>.*?)\}\}', tmpl)

        last_end = 0
        for seg in segments:

            start, end = seg.span()

            if start > last_end:
                text = tmpl[last_end:start]
                yield 'text', text

            code, data = seg.groups()

            if code:
                yield 'code', code
            if data:
                yield 'data', data

            last_end = end

        yield 'text', tmpl[last_end:]

    @lru_cache(maxsize=128)
    def _gen_code(self):
        code_gen = _CodeGenerator()

        [code_gen.handle(_type, _data)
         for _type, _data in Template._split_template(self.tmpl)]

        code_gen.end()
        return code_gen.compile()

    def render(self, eval_env=None):
        compiled_code = self._gen_code()

        env = {**BUILT_IN_ENV}

        if eval_env:
            env.update(eval_env)

        exec(compiled_code, env)

        return env['TMPL_EVAL_RESULT']
```
思路非常简单：

1. 通过 `Template._split_templte` 使用正则表达式，将模板拆分为 ('data', ...), ('text', ''), ('code', ...) 流，方便代码生成器生成代码
2. `_gen_code` 通过`_CodeGenerator`生成python代码
3. 借助python执行动态代码的能力，调用`exec` 执行代码。


### CodeGenerator
```python
class _CodeGenerator:
    def __init__(self):

        self.indent = 0
        self.code = []

        self.code.append('_result = []')
        self.code.append('_append = _result.append')

    @staticmethod
    def _is_control_statement(code):
        code = code.strip()
        control_flags = [
            lambda: code.startswith('if '),
            lambda: code.startswith('elif '),
            lambda: code.startswith('else'),
            lambda: code.startswith('try '),
            lambda: code.startswith('except '),
            lambda: code.startswith('finnaly '),
            lambda: code.startswith('with '),
            lambda: code.startswith('for '),
            lambda: code.startswith('while '),
            lambda: code.startswith('class '),
            lambda: code.startswith('def ')
        ]
        return any([is_control() for is_control in control_flags])

    def add_code(self, _code):
        if _code.strip() == 'end':
            self.indent -= 4
        else:
            self.code.append('\n')
            self.code.append(' ' * self.indent + _code.strip())

            if _CodeGenerator._is_control_statement(_code):
                self.indent += 4

    def add_text(self, text):
        self.code.append(' ' * self.indent + f'_append("""{text}""")')

    def add_data(self, data):
        self.code.append(' ' * self.indent + f'_append({data})')

    def handle(self, _type, _data):
        getattr(self, f'add_{_type}')(_data)

    def end(self):
        self.code.append('TMPL_EVAL_RESULT = "".join(_result)')

    def compile(self):
        raw_code = '\n'.join(self.code)

        if DEBUG_TEMPLATE:
            print('>>' * 10, 'start generated code', '<<' * 10)
            print(raw_code)
            print('>>' * 10, 'end generated code', '<<' * 10)

        return compile(raw_code, f'tmpl_{abs(id(self))}.py', 'exec')
```
CodeGenerator 看起来稍微复杂一点点：

1. `handle`负责依据模板拆解后的元素进行分发
2. `indent` 用来维护正确的缩进，生成格式正确的代码，遇到control结构(if/else/for...)的时候 indent + 4，遇到end: indent - 4 
3. 通过一个 _result 数组来收集数据，code 来收集代码
4. 最后需要调用 `CodeGenerator.end()`，将result的join到一个变量TMPL_EVAL_RESULT
5. 这个变量可以在`exec`完成后，在执行环境中获取, 从而得到模板渲染结果

### 模板内置支持的env
```python
BUILT_IN_ENV = {
    'os': os,
    're': re,
    'itertools': itertools,
    'functools': functools,
    'math': math
}
```

### 测试下效果

```python
def test_template():
    class Obj:
        pass

    def jfield(jtype: str, name: str, *, modifier: str = '', init_value: str = '', comment: str = ''):
        field = Obj()
        field.modifier = modifier
        field.jtype = jtype
        field.name = name
        field.initial_value = init_value
        field.comment = comment

        return field

    data = Obj()
    data.name = 'ShareConfig'
    data.fields = []
    data.fields.append(jfield('Tencent', 'tencent', modifier='private final'))
    data.fields.append(
        jfield('IWXApi', 'wxApi', modifier='private final', comment='wechat share'))

    print(Template(TMPL_BUILDER).render({'jclass': data}))
```

##### 生成的java代码

```java
private ShareConfig(Builder builder) {
    this.tencent = builder.tencent;
    this.wxApi = builder.wxApi;
}

public static class Builder {

    private Tencent tencent = null;
    private IWXApi wxApi = null;

    public Builder setTencent(Tencent tencent) {
        this.tencent = tencent;
        return this;
    }

    /**
     * wechat share
     */
    public Builder setWxapi(IWXApi wxApi) {
        this.wxApi = wxApi;
        return this;
    }

    public ShareConfig build() {
        return new ShareConfig(this);
    }
}
```

### 生成的python模板渲染代码

```python
_result = []
_append = _result.append
_append("""
""")


fields = jclass.fields
_append("""
private """)
_append(jclass.name)
_append("""(Builder builder) {
""")


for field in fields:
    _append("""    this.""")
    _append(field.name)
    _append(""" = builder.""")
    _append(field.name)
    _append(""";
""")
_append("""}

public static class Builder {

""")


for field in fields:
    _append("""    private """)
    _append(field.jtype)
    _append(""" """)
    _append(field.name)
    _append(""" = null;
""")
_append("""
""")


for field in fields:


    if field.comment:
        _append("""    /**
     * """)
        _append(field.comment)
        _append("""
     */
""")
    _append("""    public Builder set""")
    _append(field.name.title())
    _append("""(""")
    _append(field.jtype)
    _append(""" """)
    _append(field.name)
    _append(""") {
        this.""")
    _append(field.name)
    _append(""" = """)
    _append(field.name)
    _append(""";
        return this;
    }

""")
_append("""    public """)
_append(jclass.name)
_append(""" build() {
        return new """)
_append(jclass.name)
_append("""(this);
    }
}
""")
TMPL_EVAL_RESULT = "".join(_result)
```