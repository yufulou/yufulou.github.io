+++
title= "extjs4.1，能显示纵坐标中心为0点，可显示负数的area图表"
date= 2013-02-21T17:03:12+08:00
categories = ["js"]
description = ""
featured = ""
featuredalt = ""
featuredpath = ""
linktitle = ""
type = "post"
+++

效果：在一个window中，创建一个area chart，纵坐标中心为0点，负值从0点向下显示，正值向上显示，并在chart上创建蒙板，在蒙板上放置随鼠标移动的纵向基线和指示当前数据的框

效果图：
![image](/img/blog/ext4-negtive-number-chart/1.jpg)


```
Ext.onReady(function(){

        function generate_flow_data(num_count, min_num, max_num){
            var numJson = [];
            for(var i=0;i<(num_count || 100);i++){
                numJson[i] = {id:i};
                numJson[i]["up"] = get_range_number(min_num||5, max_num||50);
                numJson[i]["down"] = get_range_number(min_num||5, max_num||50);
            }
            
            function get_range_number(start_num, end_num){
                var num = Math.floor(Math.random()*(end_num - start_num) + start_num);
                return num;
            }
            return numJson;
        }
        
        function format_flow_data(numJson, max_num){
            var json_len = numJson.length;
            
            for(var i=0;i                 // 下载到0点的需求即：0 == padding+down
                numJson[i]["padding"]=-((max_num||100)-numJson[i]["down"]);
                numJson[i]["down"] = -numJson[i]["padding"];
            }
        }
        
        var json_test = generate_flow_data(100, 5, 100);
        format_flow_data(json_test, 100);
        
        var jsonData = json_test;

        var fields = ['id', 'padding', 'down', 'up'];

        var browserStore = Ext.create('Ext.data.JsonStore', {
            fields: fields,
            data: jsonData
        });

        var colors = ['rgb(254, 254, 254)',
                      'rgb(60, 133, 46)',
                      'rgb(234, 102, 17)'];

        Ext.chart.theme.Browser = Ext.extend(Ext.chart.theme.Base, {
            constructor: function(config) {
                Ext.chart.theme.Base.prototype.constructor.call(this, Ext.apply({
                    colors: colors
                }, config));
            }
        });

        var chart = Ext.create('Ext.chart.Chart', {
                id: 'chartCmp',
                xtype: 'chart',
                style: 'background:#fff',
                animate: true,
                theme: 'Browser:gradients',
                defaultInsets: 30,
                store: browserStore,
                mask: 'horizontal',
                axes: [{
                    type: 'Numeric',
                    id:"left_axes",
                    position: 'left',
                    fields: fields.slice(1),
                    title: 'Usage %',
                    grid: true,
                    decimals: 0,
                    minimum: -100,
                    maximum: 100,
                    listeners:{
                        'mouseenter':function(){
                            
                        }
                    }
                }, {
                    type: 'Category',
                    position: 'bottom',
                    fields: ['id'],
                    title: 'id'
           /*         label: {
                        renderer: function(v) {
                            return v.match(/([0-9]*)\/[0-9]*\/[0-9][0-9]([0-9]*)/).slice(1).join('/');
                        }
                    }*/
                }],
                series: [{
                    type: 'area',
                    axis: 'left',
                    xField: 'name',
                    yField: fields.slice(1),
                    style: {
                        lineWidth: 1,
                        stroke: '#666',
                        opacity: 0.86
                    }
                }]
            });
    

        var win = Ext.create('Ext.window.Window', {
            width: 800,
            height: 600,
            minHeight: 400,
            minWidth: 550,
            hidden: false,
            shadow: false,
            maximizable: false,
            title: 'What is the trend in Browser Usage?',
            renderTo: Ext.getBody(),
            layout: 'absolute',
            tbar: [{
                text: 'Save Chart',
                handler: function() {
                    Ext.MessageBox.confirm('Confirm Download', 'Would you like to download the chart as an image?', function(choice){
                        if(choice == 'yes'){
                            chart.save({
                                type: 'image/png'
                            });
                        }
                    });
                }
            }],
            items: [{
                xtype:'panel',
                id:"fit_panel",
                x:0,
                y:0,
                anchor: '100% 100%',
                layout:{
                    type:'fit',
                    align:'stretch'
                },
                items:[
                    chart
                ]
            }]
        });
                
        var chart_grid_rect = chart.chartBBox;
        var mask_panel = Ext.create("Ext.panel.Panel",{
            id:'realtime_mask_panel',
            width:chart_grid_rect.width,
            height:chart_grid_rect.height,
            x:chart_grid_rect.x+2,
            y:chart_grid_rect.y,
            border:false,
            layout:{
                type:'absolute'
            },
            bodyStyle:{
                'background':'transparent'
            },
            html:"",
            items:[{
                id:'tip_panel',
                height:50,
                width:100,
                x:"50%",
                y:0,
                bodyStyle:{
                    'background':'transparent'
                },
                html:'
abc
bcd
'
            }]
        });
        
        win.add(mask_panel);

        // 必须在afterrender之后运行
        chart.on({
            'resize':function(me){
                var chart_box = me.chartBBox;
                mask_panel.setWidth(chart_box.width);
                mask_panel.setHeight(chart_box.height);
                mask_panel.setPosition(chart_box.x+2, chart_box.y);
            }
        })
        
        var slide_line = Ext.get("slide_line");
        slide_line.hide();
        
        // getEl必须在render以后调用
        mask_panel.getEl().on({
            'mousemove':function(eobj){
                slide_line.show();
                var mask_left = mask_panel.getPosition()[0];
                var left_pos = eobj.getX();
        //        console.log(point_obj);
                slide_line.setStyle({left:(left_pos-mask_left)+'px'});
            },
            'mouseout':function(eobj){
                var mouse_left = eobj.getX();
                var mouse_top = eobj.getY();
                var mask_rect = mask_panel.getPosition();
                var mask_width = mask_panel.getWidth();
                var mask_height = mask_panel.getHeight();
                var mask_left = mask_rect[0];
                var mask_top = mask_rect[1];
                if(mouse_left<=mask_rect||mouse_left>=mask_left+mask_width||mouse_top<=mask_top||mouse_top>=mask_top+mask_height){
                    slide_line.hide();
                }
            }
        })
        
        
        var times_i = 0;
        var reload_realtime_pic = {
                run:function(){
                    var json_test = generate_flow_data(100, 5, 100);
                    format_flow_data(json_test, 100);
                    console.log(json_test);
                    browserStore.loadData(json_test);
                    times_i++;
                    if(times_i>1){
                        runner.stop(reload_realtime_pic);
                    }
                },
                interval:2000
        }
        
        var runner = new Ext.util.TaskRunner();
        runner.start(reload_realtime_pic);
    });
```