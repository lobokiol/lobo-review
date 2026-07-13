/*
*@param width 页面的宽度，可有可无
*@param url 返回首页的地址,可有可无
*当两个参数都存在的时候，请按 width,url 的顺序写
*IE6下会有按钮随滚动抖动的情况，可以在<head>标签内添加*html{background-image:url(about:blank);background-attachment:fixed;}
*author:wly
*date:2016/2/3
*/
function CreatGoToTop(width,url){
    this.arg = arguments;
    this.bw = '1000'; //页面宽度
    this.url = '';   //返回首页的url
    this.ctid = "scroll_top"; //要显示回顶部按钮的Id
    this.cssText = "position:fixed; left:50%; bottom:100px;margin-left:510px; width:55px; height:113px; _position:absolute; _bottom:auto; _margin-bottom:100px; _top:expression(eval(document.documentElement.scrollTop+document.documentElement.clientHeight-this.offsetHeight-(parseInt(this.currentStyle.marginTop,10)||0)-(parseInt(this.currentStyle.marginBottom,10)||0)));";
}
CreatGoToTop.prototype ={
    init : function(){
        if(this.arg.length==1){
            var val = this.arg[0];
            if(!isNaN(val)){
                this.bw  = val;
            }else{
                this.url = val;
            }
        }else if(this.arg.length==2){
            this.bw = this.arg[0];
            this.url = this.arg[1];
        }
    },
    goToTop : function(width,h_url){
        this.init();
        var temp ="";
        if(this.url!=''){
            temp += '<div id="newsCommentBar" style="'+this.cssText+'"><a title="返回首页" href="'+this.url+'" class="link_home" style="display:block; width:55px;height:45px; color:#858585; background:url(//t4.chei.com.cn/common/images/go-top.png) 0 -128px no-repeat ; margin-bottom:5px;padding-top:9px;text-align:center;font-size:12px;line-height:18px;text-decoration:none;">返回<br/>首页</a><a title="返回顶部" href="javascript:;" id="'+this.ctid+'" style="width:55px; height:54px;background:url(//t4.chei.com.cn/common/images/go-top.png) 0 0 no-repeat ; display:none;"></a></div>';
        }else{
            temp += '<div id="newsCommentBar" style="'+this.cssText+'"><a title="返回顶部" href="javascript:;" id="'+this.ctid+'" style="width:55px; height:54px;background:url(//t4.chei.com.cn/common/images/go-top.png) no-repeat 0 0 ; display:none;"></a></div>';
        }
        if($("#newsCommentBar").size()>0) {
            return;
        }
        $("body").append(temp);
        
        /*调整整个盒子的左边距*/
        $("#newsCommentBar").css('margin-left',(this.bw*0.5+10)+'px');
        
        /*鼠标移上去样式变化*/
        var $obj = $("#" + this.ctid);
        $(".link_home").mouseenter(function(){
            $(this).css({'background-position':'0 -192px','color':'#fff'})
        }).mouseout(function(){
            $(this).css({'background-position':'0 -128px','color':'#858585'})
        });
        $obj.mouseenter(function(){
            $(this).css('background-position','0 -64px')
        }).mouseout(function(){
            $(this).css('background-position','0 0')
        });
        
        /*是否显示回顶部按钮*/
        var showGoToTopBtn =function(){
            var top = $(window).scrollTop(); 
            if(top > 0){
                $obj.css("display" , "block");
            } else {
                $obj.css("display" , "none");
            }
        }
        
        $(window).scroll(function(){showGoToTopBtn()});
        $obj.click(function(e){
            $('html,body').animate({scrollTop: 0}, 400);
            e.preventDefault();
        })
    }
}