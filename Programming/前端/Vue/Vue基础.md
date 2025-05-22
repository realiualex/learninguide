## 说明
vue 主要是负责数据和逻辑交互，所以如果单纯只用vue的话，还是要自己写 css样式。但也可以和 ant design vue结合使用，让 ant design vue来处理css样式（或者Element Plus, Vuetify 等vue 生态的UI框架，如果是别的生态如 React, Angular，要用另外的 UI库）

## 基础语法
vue2 和vue3在创建vue对象的时候完全不同，一定要注意

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>VueProject</title>
</head>
<body>

<div id="app">
    {{message}}
    <span v-bind:title="message">鼠标悬停绑定</span>

    <h1 v-if="ok">OK</h1>
    <h1 v-else>NOK</h1>

    <h1 v-if="type==='A'">A</h1>
    <h1 v-else-if="type==='B'">B</h1>

    <li v-for="(item, index) in items">{{item.message}}--{{index}}</li>

    <button v-on:click="sayHi">click me</button>
</div>



<script src="https://unpkg.com/vue@3/dist/vue.global.js"></script>
<script>
    const app = Vue.createApp(
        {
            data() {
                return {
                    message: "yyyy",
                    ok: false,
                    type: "B",
                    items: [
                        {message: "msg1"},
                        {message: "msg2"},
                        {message: "msg3"}
                    ]
                }
            },
            methods: {
                sayHi(event)  {
                    alert(this.message)
                }
            }
        }
    )
    app.mount("#app")

    console.log(app.message)
</script>

</body>
</html>
```

