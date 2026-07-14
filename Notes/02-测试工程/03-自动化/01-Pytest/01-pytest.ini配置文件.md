```ini
[pytest]
#命令行参数
#常见：--html=./reports/report.html --reruns 2
addopts= -Vs  
#配置执行的用例位置
testpaths = ./testcases  
#配置修改默认的模块规则
python_files = test_*.py 
#配置修改默认的类规则
python_classes = Test*   
#配置修改默认的用例规则
python_functions = test_*

#配置基础路径
base_url =http://www.baidu.com
#标记
markers=
	smoke：冒烟测试用例
	product_manage：商品管理
	user_manager：用户管理
	

```