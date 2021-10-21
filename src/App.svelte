<script>

let barcode = "";

let bookInfo = [];

let contact="";

function remove(index){
		bookInfo.splice(index, 1);
		bookInfo=bookInfo;
		
}

function CopyToClipboard(){
    var r = document.createRange(); 
    r.selectNode(document.getElementById("content"));
    r.setStart(document.getElementById("content"),1);
    window.getSelection().removeAllRanges();
    window.getSelection().addRange(r);
    document.execCommand('copy');
    window.getSelection().removeAllRanges();
}




function initialized(){
	console.log("Loaded")
	 var _scannerIsRunning = false;
	
}
	
function startScanner() {
    Quagga.init({
        inputStream: {
            name: "Live",
            type: "LiveStream",
            target: document.querySelector('#scanner-container'),
            constraints: {
                width: 480,
                height: 320,
                facingMode: "environment"
            },
        },
        frequency: 4,
        decoder: {
            readers: [
                //"code_128_reader",
                "ean_reader",
                //"ean_8_reader",
                //"code_39_reader",
                //"code_39_vin_reader",
                //"codabar_reader",
                //"upc_reader",
                //"upc_e_reader",
                //"i2of5_reader"
                ],
                //SEEMS TO WORK BETTER WITHOUT THESE LAST THREE TURNED ON
                //locate: true,
                //halfsample:true,
                //patchsize: "medium",
            debug: {
                showCanvas: true,
                showPatches: true,
                showFoundPatches: true,
                showSkeleton: true,
                showLabels: true,
                showPatchLabels: true,
                showRemainingPatchLabels: true,
                boxFromPatches: {
                    showTransformed: true,
                    showTransformedBox: true,
                    showBB: true
                }
            }
        },

    }, function (err) {
            if (err) {
                console.log(err);
                return
            }

        console.log("Initialization finished. Ready to start");
        Quagga.start();

        // Set flag to is running
        _scannerIsRunning = true;
    });

            Quagga.onProcessed(function (result) {
                var drawingCtx = Quagga.canvas.ctx.overlay,
                drawingCanvas = Quagga.canvas.dom.overlay;

                if (result) {
                    if (result.boxes) {
                        drawingCtx.clearRect(0, 0, parseInt(drawingCanvas.getAttribute("width")), parseInt(drawingCanvas.getAttribute("height")));
                        result.boxes.filter(function (box) {
                            return box !== result.box;
                        }).forEach(function (box) {
                            Quagga.ImageDebug.drawPath(box, { x: 0, y: 1 }, drawingCtx, { color: "green", lineWidth: 2 });
                        });
                    }

                    if (result.box) {
                        Quagga.ImageDebug.drawPath(result.box, { x: 0, y: 1 }, drawingCtx, { color: "#00F", lineWidth: 2 });
                    }

                    if (result.codeResult && result.codeResult.code) {
                        Quagga.ImageDebug.drawPath(result.line, { x: 'x', y: 'y' }, drawingCtx, { color: 'red', lineWidth: 3 });
                    }
                }
            });


            Quagga.onDetected(function (result) {
                console.log("Barcode detected and processed : [" + result.codeResult.code + "]", result);
                barcode = result.codeResult.code;
               
               
                if(checkIfAlreadyListed()){
                    if(checkIfValidBarcode()){
                        console.log("starting api")
                        getBookInfo();


                    }

                }
                
                
                
            });
}

//returns true if not already listed, false otherwise
function checkIfAlreadyListed(){
    let i=0;
    //iterates through bookInfo checking for isbn. If isbn is found it breaks loop early preventing
    //preventing i from reaching full length value
    //if i is full length value it responds true that the isbn has not been added to list yet
    loop1: while (i< bookInfo.length){
            if(bookInfo[i].isbn == barcode){
                console.log("already recorded book");
                break loop1
            }
            else{
                i=i+1;
            }
    }
    if(i==bookInfo.length){
        return true
    }
    else{
        return false
    }

}

//returns true if valid barcode, false otherwise
function checkIfValidBarcode(){
    if (barcode.length == 10 ){
			//check valid barcode
			console.log("starting 10 digit check")
			let sum = 0;
        for (let i = 0; i < 9; i++)
        {
            let digit = barcode[i] - '0';
               console.log(digit);
            if (digit%1 != 0){
							
							console.log("not a valid isbn");
							return false
						}
                
				
            sum += (digit * (10 - i));
        }
				let last = barcode[9];
        if (last != 'X' && (last < '0' || last > '9')){
            console.log("not a valid isbn");
					return false
				}
        // If last digit is 'X', add 10
        // to sum, else add its value.
        sum += ((last == 'X') ? 10 : (last - '0'));
   
        // Return true if weighted sum
        // of digits is divisible by 11.
        if (sum % 11 == 0){
			return true	
				}
		else {
            console.log("failed 10 digit checksum");
            return false
        }	
		
			

			
			
	}

    if (barcode.length == 13 ){
			//check valid barcode does checksum of first 12 digits against the 13th
			console.log("starting 13 digit isbn check")
			let sum =0;
			for (let i = 0; i < 12; i++)
        {
            let digit = barcode[i] - '0';
              
            if (0 > digit || 9 < digit){
                console.log("not a valid barcode");
                return false;
            }
                
            if ((i+1) % 2 == 0){ sum += (digit *3)} ;     
            if ((i+1) % 2 == 1){ sum += (digit *1)} ; 
        }
			let last = barcode[12]-'0';
			console.log(sum);
			console.log(last);
		if ((10 - sum % 10) == (last) || (sum%10 == 0 && last == 0)){
					console.log("valid Barcode");
                    return true;
					
		}
        else{
            console.log("failed 13 checksum");
            return false;
        }
			
			
	}
        


    else {
        console.log("not 10 or 13 digits")
    }

}


async function getBookInfo(){
    var apiCall = "https://www.googleapis.com/books/v1/volumes?q=isbn:"+barcode;
	console.log(apiCall);
	let response = await fetch(apiCall);
	let bookJSON = await response.json();
    console.log(bookJSON);
    
    bookInfo.push({title:bookJSON.items[0].volumeInfo.title, isbn:barcode, description:bookJSON.items[0].volumeInfo.description, date:bookJSON.items[0].volumeInfo.publishedDate, author:bookJSON.items[0].volumeInfo.authors.toString(), subject: bookJSON.items[0].volumeInfo.categories.toString(), comments:"",contactInfo:""});
    console.log(bookInfo);
    bookInfo=bookInfo;
}

	
	

	
	
	
</script>



<svelte:head>
	<script  src="https://cdn.rawgit.com/serratus/quaggaJS/0420d5e0/dist/quagga.min.js" on:load={initialized}></script>
</svelte:head>



    <title></title>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
	

<body>
    <!-- Div to show the scanner -->
    <div id="scanner-container">


         
    </div>
    <p>{barcode}</p>
    <button on:click={startScanner}>scan for books</button>  
	
	
</body>




<button on:click={CopyToClipboard}>
	Copy All
</button>
 <table id=content>
  <tr>
		<th>Delete</th>
    <th>_______Title________</th>
    <th>_____Author_______</th>
    <th>Year</th>
    <th>Subject</th>
    <th>Description</th>
		<th>cause area</th>
    <th>contact</th>
    <th>Comments</th>
  </tr>
	 
	 
	
	
	 
	{#each bookInfo as book, i}
	 
	<tr>
		<td><button on:click={()=>remove(i)}>
		 x
	 </button></td>
    <td>{book.title}</td>
    <td>{book.author}</td>
    <td>{book.date}</td>
    <td>{book.subject}</td>
    <td>{book.description}</td>
		<td><div style="height:0;width:0;">
			{book.cause}
			</div><input type="text" bind:value={book.cause}></td>
		<td><div style="height:0;width:0;">
			{contact}
			</div><input type="text" bind:value={contact}></td>
		<td><div style="height:0;width:0;">
			{book.review}
			</div><input type="text" bind:value={book.review}></td>
  </tr>

{/each}
		
	
  
</table> 



 





<style>
/* In order to place the tracking correctly */
    :global(canvas.drawing, canvas.drawingBuffer){
        position: absolute;
        left: 0;
        top: 0;
    }

    table, th, td {
	position:relative;
    border: 1px solid black;

    }
</style>

