---
title: vue
date: 2018-12-20 23:11:06
tags: vue
categories: vue
---
### Vue 父子组建的通信

**父组件对子组件进行传递参数**

&nbsp;&nbsp;&nbsp;&nbsp;例子如下
```
<div id="app">
	<my-component message="来自父组件的消息"></my-component>
</div>
<script>
	Vue.component('my-component',{
		porps:['message'],
		template: `<div>{{ message }}</div>`,
	});
	var vm = new Vue({
		el: "#app"
	})
</script>
```
<!--more -->

&nbsp;&nbsp;&nbsp;&nbsp;当我们从父组件向子组件传递参数时候，子组件需要在props定义接收的参数，例如本例中的message需要在子组件中定义好。此外，由于HTML不去分大小写，我们的组件名称不可以写成驼峰，要用-分割。使用v-bind指令可以传递数组，数字，布尔值等。如果不用的话，默认为字符串。

**单项数据流**

&nbsp;&nbsp;&nbsp;&nbsp;例子如下
```
<div id="app">
	<my-component :init-count="1"></my-component>
<div>
<script>
	Vue.component('my-component',{
		props:['initCount'],
		template:`<div>{{ count }}</div>`,
		data: function (){
			return {
				count: this.initCount
			}
		}
	});
	var vm = new Vue({
		el: "#app"	
	})
</script>
```
&nbsp;&nbsp;&nbsp;&nbsp;在这里我们在子组件的template中声明了一个count数据，而我们子组件的data定义这个count使用的是父组件传递的initCount值，当我们从父组件获取到initCount的值的时候，就与之无关了，我们维护count即可，无需在操作父组件传递的initCount。

**子组件对父组件进行传递参数**

&nbsp;&nbsp;&nbsp;&nbsp;例子如下
```
<div id="app">
	<p>总数:{{ total }}</p>
	<my-component @increase="handleGetTotal" @reduce="handleGetTotal"></my-component>
</div>
<script>
	Vue.component('my-component',{
		template:`
		<div>
			<button @click="handleIncrease">+1</button>
			<button @click="handleReduce">-1<button>
		</div>`,
		data: function (){
			return {
				counter: 0	
			}
		},
		methods: {
			handleIncrease: function (){
				this.counter++;
				this.$emit('increase', this.counter)
			},
			handleReduce: function (){
				this.counter--;
				this.$emit('reduce', this.counter)
	
	        }
		}
	});
	var vm = new Vue({
		el: "#app",
		data: {
			total: 0
		},
		methods: {
			handleGetTotal: function (total){
				this.total = total
		}
	}
});
</script>
```
&nbsp;&nbsp;&nbsp;&nbsp;从上面的例子可以发现，子组件向父组件传递参数通过``$emit``的方式，通过``$emit``将子组件的值传入了increase和reduce的方法之中,然后父组件的方法在调用传递的值。
