# productfeeds
<?php
session_start();
ob_start(); 
ini_set('display_errors', 1);
require_once('config.php');
require_once("shopify.php");
require_once('functions.php');
//require_once('function.php');
//echo $httpReferer = isset($_SERVER['HTTP_REFERER']) ? $_SERVER['HTTP_REFERER'] : null;
//die();

if(isset($_GET['shop'])){
	$STORE_URL = $_GET['shop'];
	if(isset($_SESSION['shop'])){
		if($_SESSION['shop'] == $STORE_URL){
			$response = checkShopExists($STORE_URL);
			if($response['status'] == 1){
				$token = $_SESSION['token'] = $response['result']['token'];
				$_SESSION['Id'] = $response['result']['id'];
			}
			else{
				$token = "";
			}
		}else{
			unset($_SESSION);
			$response = checkShopExists($STORE_URL);
			if($response['status'] == 1){
				$token = $_SESSION['token'] = $response['result']['token'];
				$_SESSION['Id'] = $response['result']['id'];
				$_SESSION['shop'] = $STORE_URL;
			}
			else{
				$token = "";
			}
		}
	}else{
		$response = checkShopExists($STORE_URL);
		if($response['status'] == 1){
			$token = $_SESSION['token'] = $response['result']['token'];
			$_SESSION['Id'] = $response['result']['id'];
			$_SESSION['shop'] = $STORE_URL;
		}
		else{
			$token = "";
		}
	}
}
else{
	if(isset($_SESSION['shop'])){
		$response = checkShopExists($_SESSION['shop']);
		if($response['status'] == 1){
			$token = $_SESSION['token'] = $response['result']['token'];
			$_SESSION['Id'] = $response['result']['id'];
		}
		else{
			$token = "";
		}
	}
	else{
		$token = "";
		echo "Shop url Is missing. Please try again";
		die();
	}
}

if(isset($token) && $token != ""){
	$config =  array('client_Id' => APIKEY,
					 'redirect_uri' => REDIRECT_URL,
					 'client_Secret' =>  SECRET,
					'url'=> $_SESSION['shop'],
				);
	$productFeed = new shopify($config);
	$response = $productFeed->setAccessToken($token);
	if($response['status']){
		//echo $response['result'];
	}else{
		echo $response['error'];
		die();
	}
}else{ 
	if(isset($_GET['code']) && isset($_GET['shop'])){
		$config = array('client_Id' => APIKEY,
						'code' => $_GET['code'],
						'redirect_uri' => REDIRECT_URL,
						'client_Secret' =>  SECRET,
						'url'=> $STORE_URL,
				);		
	}
	else{
		$config  = 	array('client_Id' => APIKEY,
						'redirect_uri' => REDIRECT_URL,
						'url'=> $STORE_URL,
					);
		}
		$productFeed = new shopify($config);
		$response = $productFeed->getAccessToken();
		if($response['status']){
			$data = array('token'=>$response['token'], 'shop'=>$STORE_URL);
			$response = createAccount($data);
			if($response['status']){
				$response = $productFeed->registerShopifyAppUninstallWebhook();
				if(!$response['status']){
					echo $response['error'];
					die();
				}
			}
			else{
				echo $response['error'];
				die();
			}
		}else{
			echo $response['error'];
			die();
		}
}
################ Token Get Process End #####################
$response = $productFeed->getCollection();
if($response['status']){
	$product_collections = $response['result'];
	$collection_Id = $response['IDs'];
}
else{
	echo $response['error'];
	die();
}
	$response = getCollections($_SESSION['shop']);
	if($response['status']){
		$collection_Entry = $response['result'];
	}
	if(isset($collection_Entry)){
		$collection = array_diff( $collection_Id, $collection_Entry);
	}else{
		$collection = $collection_Id;
	}
?>

<!--################ product id Get Process End #####################-->


<html>
	<head>
	
<script src="http://code.jquery.com/jquery-1.10.1.min.js"></script>

		<script type="text/javascript" src="./js/jquery.js"></script>
		<script type="text/javascript" src="./js/DataSet.js"></script>
		<script type="text/javascript" src="./js/sweetalert.min.js"></script>
		<script type="text/javascript">
			$(document).ready(function(){
				$('#myTable').DataTable({
					 "lengthChange": false,
					  "searching": false
				}); 
				
				$("#checkAllBox").click(function (){
					$(".checkBoxClass").attr('checked', this.checked)
				});
				
				var table =  $('#myTable1').DataTable(
				{"scrollY":"400px", "scrollCollapse": true});
				$("#button").click(function(){
					var feedName = $("#coll").val();
					if(feedName == ""){
						swal({
							title: "Feed Name Missing",
							text: "Please Enter Feed name",
							timer: 2000,
							showConfirmButton: false
						});
						return false;
						$("#coll").focus();
					}
					var data = table.$('input').serialize();
				$.ajax({
						type: "POST",
						url: "process2.php",
						data: data+'&FeedName='+feedName,
						async: false,
						success: function(data){
							if(data){
								swal({
								  title: "Success",
								  text: "Feed Created Successfully",
								  timer: 1000,
								  showConfirmButton: false
								});
								location.reload();
								$(".modalDialog").css({'opacity':0,'pointer-events':'none'});
								$('.tab').find('input[type=checkbox]:checked').removeAttr('checked');
								$("#coll").val("");
							}
							else{
								$('.tab').find('input[type=checkbox]:checked').removeAttr('checked');
								$("#coll").val("");
								swal({
									title: "Error",
									text: "Please select products",
									timer: 1500,
									showConfirmButton: false
								});
								
							}
						},
						error: function(){
							$('.tab').find('input[type=checkbox]:checked').removeAttr('checked');
							$("#coll").val("");
							swal({
							  title: "Error",
							  text: "Please try again",
							  timer: 1000,
							  showConfirmButton: false
							});
						}
					});
					
					
				});
			});
			
	function closeModal(e){
				e.preventDefault();
				$('.tab').find('input[type=checkbox]:checked').removeAttr('checked');
				$("#coll").val("");
				$(".modalDialog").css({'opacity':0,'pointer-events':'none'});
				
			} 
	function feeds(){
				var data = "";
				<?php
					if(is_array($collection) && !empty($collection)){
						foreach($collection as $coll){
							?>
							data += "<h3 onclick='check(<?php echo $product_collections[$coll]['id'];?>,\"<?php echo $product_collections[$coll]['title'];?>\");'>" + '<?php echo $product_collections[$coll]['title'];?>' + "</h3>"
							<?php
						}
						?>
						data = 	"<ul>"+ data +"</ul>";
						<?php
					}
					else{
						?>
						data = "Sorry, No Feed avaialble to Generate!";
						<?php
					}
				?>
				swal({
					title: "Available Feeds",
					text: data,
					showCancelButton: true,
					showConfirmButton: false,
					html: true
				});
			}
			
			function customFeed(){
				swal.close();
				$(".modalDialog").css({'opacity':1,'pointer-events':'auto'});
			}
			function feedss(){
				
					data = "<h3 onclick='feeds();'>" + '<?php echo '<a href="#" class="myButton">Create Collection Feed</a>'; ?>' + "</h3><br><h3 onclick='customFeed();'>" + '<?php echo '<a href="#" class="myButton">Create Custom Feed</a>'; ?>' + "</h3>"
					
				swal({
					title: "Select Feed Source",
					text: data,
					showCancelButton: true,
					showConfirmButton: false,
					html: true
				});
			}
			
			function check(id, title){
				swal({
				  title: "Are you sure to Create "+ title +" Feed?",
				  type: "warning",
				  showCancelButton: true,
				  confirmButtonColor: "#DD6B55",
				  confirmButtonText: "Yes, Create Feed!",
				  cancelButtonText: "No, cancel please!",
				  closeOnConfirm: false,
				  closeOnCancel: false
				},
				function(isConfirm){
				  if (isConfirm) {
				$.post("process.php",{id:id,method:'insert'},function(data){
						if(data == 1){
							swal({
							  title: "Success",
							  text: "Feed Created Successfully",
							  timer: 800,
							  showConfirmButton: false
							});
							location.reload(); 
						}
					});
				  } else {
						swal("Cancelled", "You have cancelled it", "error");
				  }
				});
			}

			function deleteCollection(id, shop, type){
				swal({
					title: "Are you sure To delete?",
					type: "warning",
					showCancelButton: true,
					confirmButtonColor: "#DD6B55",
					confirmButtonText: "Yes, delete it!",
					closeOnConfirm: false
				},
				function(){
					$.post("process.php",{id:id,shop:shop,method:'delete',type:type},function(data){
						if(data == 1){
							swal({
							  title: "Deleted",
							  text: "Feed Deleted Successfully",
							  timer: 800,
							  showConfirmButton: false
							});
							location.reload();
						}
						else{
							swal("", "Error in Deletion", "error");
						}
					});
				});
			}

</script>
		<link rel="stylesheet" type="text/css" href="./css/DataSet.css">
		<link rel="stylesheet" type="text/css" href="./css/sweetalert.css">
		<link rel="stylesheet" type="text/css" href="./css/feed.css">
		<style>
			td img{height: 54px;width: 53px;}
			.feed {height: 38px;margin-left: 244px;width: 175px;}
			.modalDialog {
				background: rgba(0, 0, 0, 0.8) none repeat scroll 0 0;
				bottom: 0;
				font-family: Arial,Helvetica,sans-serif;
				left: 0;
				opacity: 0;
				pointer-events: none;
				position: fixed;
				right: 0;
				top: 0;
				transition: opacity 400ms ease-in 0s;
				z-index: 99999;
			}
			.modalDialog:target {
				opacity: 1;
				pointer-events: auto;
			}
			.modalDialog > div {
				background: rgba(0, 0, 0, 0) linear-gradient(#fff, #999) repeat scroll 0 0;
				border-radius: 10px;
				margin: 1% auto;
				padding: 9px 20px 100px;
				position: relative;
				width: 1000px;
			}
	.thead1 > th {
				padding: 65px;
			}
			
	.close {
				background: #606061;
				color: #FFFFFF;
				line-height: 25px;
				position: absolute;
				right: 0px;
				text-align: center;
				top: -1px;
				width: 24px;
				text-decoration: none;
				font-weight: bold;
				-webkit-border-radius: 12px;
				-moz-border-radius: 12px;
				border-radius: 12px;
				-moz-box-shadow: 1px 1px 3px #000;
				-webkit-box-shadow: 1px 1px 3px #000;
				box-shadow: 1px 1px 3px #000;
			}

.close:hover { background: #00d9ff; }
			
.myButton {
	background:-webkit-gradient(linear, left top, left bottom, color-stop(0.05, #7892c2), color-stop(1, #476e9e));
	background:-moz-linear-gradient(top, #7892c2 5%, #476e9e 100%);
	background:-webkit-linear-gradient(top, #7892c2 5%, #476e9e 100%);
	background:-o-linear-gradient(top, #7892c2 5%, #476e9e 100%);
	background:-ms-linear-gradient(top, #7892c2 5%, #476e9e 100%);
	background:linear-gradient(to bottom, #7892c2 5%, #476e9e 100%);
	filter:progid:DXImageTransform.Microsoft.gradient(startColorstr='#7892c2', endColorstr='#476e9e',GradientType=0);
	background-color:#7892c2;
	-moz-border-radius:8px;
	-webkit-border-radius:8px;
	border-radius:8px;
	display:inline-block;
	cursor:default;
	color:#ffffff;
	font-family:Arial;
	font-size:19px;
	font-weight:bold;
	padding:9px 23px;
	text-decoration:none;
	text-shadow:0px 1px 0px #283966;
}
.myButton:hover {
	background:-webkit-gradient(linear, left top, left bottom, color-stop(0.05, #476e9e), color-stop(1, #7892c2));
	background:-moz-linear-gradient(top, #476e9e 5%, #7892c2 100%);
	background:-webkit-linear-gradient(top, #476e9e 5%, #7892c2 100%);
	background:-o-linear-gradient(top, #476e9e 5%, #7892c2 100%);
	background:-ms-linear-gradient(top, #476e9e 5%, #7892c2 100%);
	background:linear-gradient(to bottom, #476e9e 5%, #7892c2 100%);
	filter:progid:DXImageTransform.Microsoft.gradient(startColorstr='#476e9e', endColorstr='#7892c2',GradientType=0);
	background-color:#476e9e;
}

.myButton:active {
	position:relative;
	top:1px;
}

.sub-btn-sty {
	-moz-box-shadow: 0px 10px 14px -7px #3e7327;
	-webkit-box-shadow: 0px 10px 14px -7px #3e7327;
	
	background:-webkit-gradient(linear, left top, left bottom, color-stop(0.05, #77b55a), color-stop(1, #72b352));
	background:-moz-linear-gradient(top, #77b55a 5%, #72b352 100%);
	background:-webkit-linear-gradient(top, #77b55a 5%, #72b352 100%);
	background:-o-linear-gradient(top, #77b55a 5%, #72b352 100%);
	background:-ms-linear-gradient(top, #77b55a 5%, #72b352 100%);
	background:linear-gradient(to bottom, #77b55a 5%, #72b352 100%);
	filter:progid:DXImageTransform.Microsoft.gradient(startColorstr='#77b55a', endColorstr='#72b352',GradientType=0);
	background-color:#77b55a;
	-moz-border-radius:4px;
	-webkit-border-radius:4px;
	border-radius:4px;
	border:1px solid #4b8f29;
	display:inline-block;
	cursor:default;
	color:#ffffff;
	font-family:Arial;
	font-size:13px;
	font-weight:bold;
	padding:3px 12px;
	text-decoration:none;
	text-shadow:0px 1px 0px #5b8a3c;
}
.sub-btn-sty:hover {
	background:-webkit-gradient(linear, left top, left bottom, color-stop(0.05, #72b352), color-stop(1, #77b55a));
	background:-moz-linear-gradient(top, #72b352 5%, #77b55a 100%);
	background:-webkit-linear-gradient(top, #72b352 5%, #77b55a 100%);
	background:-o-linear-gradient(top, #72b352 5%, #77b55a 100%);
	background:-ms-linear-gradient(top, #72b352 5%, #77b55a 100%);
	background:linear-gradient(to bottom, #72b352 5%, #77b55a 100%);
	filter:progid:DXImageTransform.Microsoft.gradient(startColorstr='#72b352', endColorstr='#77b55a',GradientType=0);
	background-color:#72b352;
}
.sub-btn-sty:active {
	position:relative;
	top:1px;
}

.txt-box-sty{border: 1px solid #4b8f29;
    border-radius: 4px;
	height:27;
	margin-top: 9px;
    padding-left: 5px; margin-bottom: 10px;padding-bottom: 5px;}
	#myTable1_info {margin-left:0px;}
	
.text-select{
	     text-align:left;
		 padding-left: 22px !important;
	}
.text-image{
	     text-align:right;
		  padding-right: 24px !important;
	}
.img-right{
	text-align: right;
	padding-right: 26px !important;
	}
.chk-box {
    padding-left: 28px !important;
	}
	
	</style>
	</head>
	<body>
		<div class="main">
			<div class="header"></div>
			    <div class="body">
					<p>
						<input type="image" class="feed" src="./images/button (3).png" onclick="feedss();">
						<div id="openModal" class="modalDialog">
						<!--a href="#close" title="Close" class="close">X</a>
						<!--a href="" onclick="closeModel()" class="close" >X</a-->
						<div class="tab" style="height:78%; overflow-x: hidden; overflow-y: hidden;">
						<a href="#" onclick="closeModal(event);"  class="close" >X</a>
						<!--div class="test" style="width:100%;text-align:center; height:425px; overflow-y: scroll;"-->
						<form method ="POST">
						
						<div id="output"></div>
						<div class="submit1" align="center">
						Enter Name of Custom Feed&nbsp;&nbsp;&nbsp;<input type="text" name="collection" id="coll" placeholder="Feed Name" class ="txt-box-sty"/>&nbsp;&nbsp;&nbsp;<input type="button" class ="sub-btn-sty" id="button" value="Submit"/>
						</div>
						      
								<table id="myTable1" class="display1" cellspacing="0" >
									<!--input type="checkbox" id="checkAllBox" class = "ck-all-box"-->
									<thead>
										<tr class="thead">
											<th class="text-select">Select</th>
											<th>Title</th>
											<th class="text-image">Image</th>
											
										</tr>
										</thead>
											<tbody>
											
												<?php 
												$response = $productFeed->getAllProuctData();
												if($response['status']){
													$response = $productFeed->parseProductData($response['products']);
													foreach($response['data'] as $prod){
														?>
														<tr><td class="chk-box"><input type="checkbox" class="checkBoxClass " name="ids[]" value="<?php echo $prod['id'];?>"></td><td><?php echo $prod['title'];?></td><td class="img-right"><img src="<?php echo $prod['image_link'];?>" width="60px" height="80px"></td></tr>
														<?php
													} ?>												
													<?php
												}
												else{
													echo $response['error'];
												}
												 ?>
											</tbody>
										</table>
									</form>
								</div>
							<!--/div-->
						</div>
					</p>
					<?php 
					$resp = getCustomCollection();
					//echo '<pre>';
					//print_r($resp);
					if(isset($collection_Entry) || !empty($resp['result'])){
						?>
						<table id="myTable" class="display" cellspacing="0" style="width:60%;text-align:center;">
							<thead>
								<tr class="thead">
									<th>Title</th>
									<th>Description</th>
									<th>Feed Link</th>
									<th>Action</th>
								</tr>
							</thead>
							<tbody>
								<?php
							if(isset($collection_Entry)){
								foreach($collection_Entry as $collect){
									?>
									<tr>
										<td><?php echo $product_collections[$collect]['title'];?></td>
										<td><?php echo $product_collections[$collect]['desc'];?></td>
										<td><a href="collectionFeed.php?type=collection&id=<?php echo $product_collections[$collect]['id'];?>&shop=<?php echo $_SESSION['shop'];?>" target="_blank">Generate Feed</a></td>
										<td><img onclick="deleteCollection('<?php echo $product_collections[$collect]['id'];?>', '<?php echo $_SESSION['shop'];?>','collection');" src="./images/delete.png"></td>
									</tr>
								    <?php
								} 
							}
							if(!empty($resp['result'])){
								foreach($resp['result'] as $res){
									?>
									<tr>
										<td><?php echo $res['name'];?></td>
										<td><?php echo  $res['name'];?></td>
										<td><a href="collectionFeed.php?type=custom&id=<?php echo $res['id'];?>&shop=<?php echo $_SESSION['shop'];?>" target="_blank">Generate Feed</a></td>
										<td><img onclick="deleteCollection('<?php echo $res['id'];?>', '<?php echo $_SESSION['shop'];?>','custom');" src="./images/delete.png"></td>
									</tr>
								    <?php
								} 
							}
								?>
							</tbody>
						</table>
						<?php
					}
					else{
						?>
						<div>No Feed Yet Created. Click on Create feed Button to generate feeds.</div>
						<?php
					} ?>
				</div>
        </div>
    </body>
</html>
