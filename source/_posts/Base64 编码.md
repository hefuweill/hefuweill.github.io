---
title: Base64 编码
date: 2020-08-14 19:52:23
category: 计算机网络
---

## 前言

本文主要用于记录 Base64 编码的基本概念、原理以及其在实际开发中的作用。<!-- more -->

## 基本概念

Base64 **编码**算法是用于将**二进制数据**转换成由 **64 个可读字符**组成的字符串的编码算法，长度不定。

注1：编码的定义为将数据从一种格式转换为另一种格式，并且还能转换为原格式的一种转换方式。

注2：广义上的二进制数据，表示的是任何计算机的数据，狭义上的二进制数据表示的是除文本以外的所有数据比如：图片、音乐、视频等。对于 Base64 编码而言其代表的是**广义**的。

注3：64 个字符分别是 A ~ Z、a ~ z、0 ~ 9、+、/ 。

## 编码原理

Base64 编码会将中的二进制数据每六位替换为一个字符，比如原文为 Man，其对应 3 个字节，如下图：

<table class="table3">
<tbody><tr>
<th scope="row">文本
</th>
<td colspan="8" align="center"><b>M</b>
</td>
<td colspan="8" align="center"><b>a</b>
</td>
<td colspan="8" align="center"><b>n</b>
</td></tr>
<tr>
<th scope="row">ASCII编码
</th>
<td colspan="8" align="center">77
</td>
<td colspan="8" align="center">97
</td>
<td colspan="8" align="center">110
</td></tr>
<tr>
<th scope="row">二进制位
</th>
<td>0</td>
<td>1</td>
<td>0</td>
<td>0</td>
<td>1</td>
<td>1</td>
<td>0</td>
<td>1</td>
<td>0</td>
<td>1</td>
<td>1</td>
<td>0</td>
<td>0</td>
<td>0</td>
<td>0</td>
<td>1</td>
<td>0</td>
<td>1</td>
<td>1</td>
<td>0</td>
<td>1</td>
<td>1</td>
<td>1</td>
<td>0
</td></tr>
<tr>
<th scope="row">索引
</th>
<td colspan="6" align="center">19
</td>
<td colspan="6" align="center">22
</td>
<td colspan="6" align="center">5
</td>
<td colspan="6" align="center">46
</td></tr>
<tr>
<th scope="row">Base64编码
</th>
<td colspan="6" align="center"><b>T</b>
</td>
<td colspan="6" align="center"><b>W</b>
</td>
<td colspan="6" align="center"><b>F</b>
</td>
<td colspan="6" align="center"><b>u</b>
</td></tr></tbody>
</table>

那么为什么 19 就对应于 T、22 就对应于 W 呢？其实它拥有一个码表，根据码表一一替换，如下图：

<table style="text-align:center">
<tbody><tr>
<th scope="col">数值</th>
<th scope="col">字符
</th>
<td rowspan="18">&nbsp;
</td>
<th scope="col">数值</th>
<th scope="col">字符
</th>
<td rowspan="18">&nbsp;
</td>
<th scope="col">数值</th>
<th scope="col">字符
</th>
<td rowspan="18">&nbsp;
</td>
<th scope="col">数值</th>
<th scope="col">字符
</th></tr>
<tr>
<td>0</td>
<td>A</td>
<td>16</td>
<td>Q</td>
<td>32</td>
<td>g</td>
<td>48</td>
<td>w
</td></tr>
<tr>
<td>1</td>
<td>B</td>
<td>17</td>
<td>R</td>
<td>33</td>
<td>h</td>
<td>49</td>
<td>x
</td></tr>
<tr>
<td>2</td>
<td>C</td>
<td>18</td>
<td>S</td>
<td>34</td>
<td>i</td>
<td>50</td>
<td>y
</td></tr>
<tr>
<td>3</td>
<td>D</td>
<td>19</td>
<td>T</td>
<td>35</td>
<td>j</td>
<td>51</td>
<td>z
</td></tr>
<tr>
<td>4</td>
<td>E</td>
<td>20</td>
<td>U</td>
<td>36</td>
<td>k</td>
<td>52</td>
<td>0
</td></tr>
<tr>
<td>5</td>
<td>F</td>
<td>21</td>
<td>V</td>
<td>37</td>
<td>l</td>
<td>53</td>
<td>1
</td></tr>
<tr>
<td>6</td>
<td>G</td>
<td>22</td>
<td>W</td>
<td>38</td>
<td>m</td>
<td>54</td>
<td>2
</td></tr>
<tr>
<td>7</td>
<td>H</td>
<td>23</td>
<td>X</td>
<td>39</td>
<td>n</td>
<td>55</td>
<td>3
</td></tr>
<tr>
<td>8</td>
<td>I</td>
<td>24</td>
<td>Y</td>
<td>40</td>
<td>o</td>
<td>56</td>
<td>4
</td></tr>
<tr>
<td>9</td>
<td>J</td>
<td>25</td>
<td>Z</td>
<td>41</td>
<td>p</td>
<td>57</td>
<td>5
</td></tr>
<tr>
<td>10</td>
<td>K</td>
<td>26</td>
<td>a</td>
<td>42</td>
<td>q</td>
<td>58</td>
<td>6
</td></tr>
<tr>
<td>11</td>
<td>L</td>
<td>27</td>
<td>b</td>
<td>43</td>
<td>r</td>
<td>59</td>
<td>7
</td></tr>
<tr>
<td>12</td>
<td>M</td>
<td>28</td>
<td>c</td>
<td>44</td>
<td>s</td>
<td>60</td>
<td>8
</td></tr>
<tr>
<td>13</td>
<td>N</td>
<td>29</td>
<td>d</td>
<td>45</td>
<td>t</td>
<td>61</td>
<td>9
</td></tr>
<tr>
<td>14</td>
<td>O</td>
<td>30</td>
<td>e</td>
<td>46</td>
<td>u</td>
<td>62</td>
<td>+
</td></tr>
<tr>
<td>15</td>
<td>P</td>
<td>31</td>
<td>f</td>
<td>47</td>
<td>v</td>
<td>63</td>
<td>/
</td></tr></tbody>
</table>

不过这只是在一般情况下，如果要编码的字节数不能被 3 整除，最后会多出 1 个或 2 个字节，这时候的处理方式为先使用 0 补足，使其能够被 3 整除，然后再进行 Base64 的编码。在编码后的Base64文本后加上一个或两个 `=` 号，代表补足的字节数，如下图所示：

<table class="table3">
<tbody><tr>
<th scope="row">文本（1 Byte）
</th>
<td colspan="8" align="center"><b>A</b>
</td>
<td colspan="8" align="center"><i></i>
</td>
<td colspan="8" align="center"><i></i>
</td></tr>
<tr>
<th scope="row">二进制位
</th>
<td>0</td>
<td>1</td>
<td>0</td>
<td>0</td>
<td>0</td>
<td>0</td>
<td>0</td>
<td>1</td>
<td>0<i></i></td>
<td>0<i></i></td>
<td>0<i></i></td>
<td>0<i></i></td>
<td>0<i></i></td>
<td>0<i></i></td>
<td>0<i></i></td>
<td>0<i></i></td>
<td>0<i></i></td>
<td>0<i></i></td>
<td>0<i></i></td>
<td>0<i></i></td>
<td>0<i></i></td>
<td>0<i></i></td>
<td>0<i></i></td>
<td>0<i></i>
</td></tr>
<tr>
<th scope="row">二进制位（补0）
</th>
<td>0</td>
<td>1</td>
<td>0</td>
<td>0</td>
<td>0</td>
<td>0</td>
<td>0</td>
<td>1</td>
<td><b>0</b></td>
<td><b>0</b></td>
<td><b>0</b></td>
<td><b>0</b></td>
<td>0<i></i></td>
<td>0<i></i></td>
<td>0<i></i></td>
<td>0<i></i></td>
<td>0<i></i></td>
<td>0<i></i></td>
<td>0<i></i></td>
<td>0<i></i></td>
<td>0<i></i></td>
<td>0<i></i></td>
<td>0<i></i></td>
<td>0<i></i>
</td></tr>
<tr>
<th scope="row">Base64编码
</th>
<td colspan="6" align="center"><b>Q</b>
</td>
<td colspan="6" align="center"><b>Q</b>
</td>
<td colspan="6" align="center">=<i></i>
</td>
<td colspan="6" align="center">=<i></i>
</td></tr>
<tr>
<th scope="row">文本（2 Byte）
</th>
<td colspan="8" align="center"><b>B</b>
</td>
<td colspan="8" align="center"><b>C</b>
</td>
<td colspan="8" align="center"><i></i>
</td></tr>
<tr>
<th scope="row">二进制位
</th>
<td>0</td>
<td>1</td>
<td>0</td>
<td>0</td>
<td>0</td>
<td>0</td>
<td>1</td>
<td>0</td>
<td>0</td>
<td>1</td>
<td>0</td>
<td>0</td>
<td>0</td>
<td>0</td>
<td>1</td>
<td>1</td>
<td>0<i></i></td>
<td>0<i></i></td>
<td>0</td>
<td>0</td>
<td>0</td>
<td>0</td>
<td>0</td>
<td>0
</td></tr>
<tr>
<th scope="row">二进制位（补0）
</th>
<td>0</td>
<td>1</td>
<td>0</td>
<td>0</td>
<td>0</td>
<td>0</td>
<td>1</td>
<td>0</td>
<td>0</td>
<td>1</td>
<td>0</td>
<td>0</td>
<td>0</td>
<td>0</td>
<td>1</td>
<td>1</td>
<td><b>0</b></td>
<td><b>0</b></td>
<td>0</td>
<td>0</td>
<td>0</td>
<td>0</td>
<td>0</td>
<td>0
</td></tr>
<tr>
<th scope="row">Base64编码
</th>
<td colspan="6" align="center"><b>Q</b>
</td>
<td colspan="6" align="center"><b>k</b>
</td>
<td colspan="6" align="center"><b>M</b>
</td>
<td colspan="6" align="center">=<i></i>
</td></tr></tbody>
</table>

很明显如果原字节数与 3 的模为 1，也就是多余 8 位，Base64 会将其补足  24 位，因此最终会多两个 ```=``` 号，如果模为 2，也就是多余 16 位 Base64 将其补足 24 位，因此最终会多一个 ```=``` 号。

<style>
    .table3 th:nth-of-type(1){
    	width: 20%;
    }
    .table3 th:nth-of-type(2){
    	width: 20%;
    }
    .table3 th:nth-of-type(3){
    	width: 60%;
    }
    td, th {
        text-align:center;
        border: 1px solid gray;
    }
</style>

## 基本用途

用于在只能接收文本信息的地方传输二进制数据，比如客户端通过 WebView 传输视频给 H5，或者 IM 聊天发送图片给对方（也可以客户端先上传图片再发送 url 给对方）。

## 性能对比

Base64 编码不提供安全性，因为算法是固定的，人人都可以获取到源数据。

Base64 编码是低效的，只在**非用不可**的情况下使用，原因是其将 3/4 个字节转换为 1 个字节，增加了传输成本以及运算成本。

## 扩展知识

Base64 编码还有一个相近的编码那就是 Base58 编码主要用于生成比特币的钱包地址，其去除了 O、0、I、l、+、\，其中前面 4 个字符是由于其太相似了容易搞混，最后去除 +、\ 是由于不方便双击复制文本。