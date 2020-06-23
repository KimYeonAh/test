
#  출석체크 

출석체크 기능은 공부를 하기 위해 책상에 앉을 수 있도록 도와주는 기능이다. <br>
Time Table에서 시간표를 설정한 후 지정한 시간에 알람이 울리면 출석체크를 실행한다.<br>
출석체크 방법에는 세 가지가 있다.
- firebase ML Kit를 이용한 사물(책상)인식
- firebase ML Kit를 이용한 Text 인식 
- OpenCV를 이용한 Color Histogram Image Matching

우선, firebase ML Kit와 openCV를 사용하기 위해 (2번)내용을 수행해야 한다.


## 출석체크 방법 선택 

**AttendanceCheckActivity.java**

알람이 울릴 때, 어떤 방식으로 출석체크를 할지 선택하면서, 선택한 방식으로 출석체크 후 출석과 결석을 판단해주는 Activity이다.<br>
총 네 가지의 button이 존재한다.

- button을 클릭하면 해당 Activity로 이동하여 출석체크 하는 동안 알람이 일시정지된다. 

아래는 각 button들의 `onClickListener()`이다.
~~~java
btnCheck = (Button)findViewById(R.id.btnCheck);
btnCheck.setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View v) {
        mediaPause();
        startActivityForResult(new Intent(getApplicationContext(), ImageLabelActivity.class), LABEL_ACTIVITY);
    }
});

btnTextCheck = (Button)findViewById(R.id.btnTextCheck);
btnTextCheck.setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View v) {
        mediaPause();
        textRecognition();
    }
});

btnSkip = (Button)findViewById(R.id.btnSkip);
btnSkip.setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View v) {
        mediaPause();
        dialogSkip();
    }
});

btnOpencv = (Button)findViewById(R.id.btnOpencv);
btnOpencv.setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View v) {
        mediaPause();
        startActivityForResult(new Intent(getApplicationContext(), ImageMatchingActivity.class), IMAGE_MATCHING_ACTIVITY);
    }
});
}
~~~

랜덤으로 하나의 영어단어를 함께 넘겨주기위해 `String.xml` 에 영어단어 10가지를 추가하여 배열에 넣어주었다.
~~~java
randomText = getResources().getStringArray(R.array.random_text);  
rnd = new Random();
~~~

>String.xml
>~~~java
><string-array name="random_text">  
> <item>passion</item>  
> <item>wish</item>  
> <item>aspiration</item>  
> <item>peace</item>  
> <item>blossom</item>  
> <item>sunshine</item>  
> <item>cherish</item>  
> <item>smile</item>  
> <item>family</item>  
> <item>rainbow</item>  
></string-array>
>~~~

Text 인식을 이용한 출석체크 버튼을 눌렀을 경우 실행되는 `textRecognition()` method 이다.<br>
랜덤으로 randomText배열에 들어가 있는 영어단어 하나를 함께 **TextRecognitionActivity**로 보내준다. 
~~~java
public void textRecognition(){  
	Intent intent = new Intent(this, TextRecognitionActivity.class );  
	int num = rnd.nextInt(9);  
	intent.putExtra("English", randomText[num]);  
  
	startActivityForResult(intent, TEXT_ACTIVITY);  
}
~~~

Skip button을 제외한 각 버튼들을 클릭하면, `startActivityForResult()`를 통해 Activity마다 다른 requestCode와 함께 해당 Activity로 넘겨준다. 아래는 각 Activity의 requestCode를 정의해준 것이다.
~~~java
// 출석체크시 Activity 구분을 위한 requestCode  
final int LABEL_ACTIVITY = 1;  //사물 인식 Activity
final int TEXT_ACTIVITY = 2;  //Text 인식 Activity
final int IMAGE_MATCHING_ACTIVITY = 3; // Image Matching Activity
~~~


`startActivityForResult()`를 사용하여 다른 Activity를 실행해줬을 경우, `onActivityResult()`를 통해 Activity의 결과를 가져와 출석체크 출결여부를 결정한다.<br>
Activity의 구분은 위에서 함께 넘겨준 requestCode로 구분할 수 있다.

- 출석체크 완료 시, 출석 count를 증가시키고 알람과 Activity를 꺼준다.
- 출석체크 실패 시, 출석체크를 다시 수행할 수 있게 일시정지 되었던 알람이 다시 울린다. 
~~~java
@Override
protected void onActivityResult(int requestCode, int resultCode, @Nullable Intent data) {
    super.onActivityResult(requestCode, resultCode, data);

    switch (requestCode) {
        case LABEL_ACTIVITY:

        String label = data.getStringExtra("labeling");

        if(label.equals("Desk") || label.equals("Table") ){
            Toast.makeText(this, "Label 출석체크 완료 : "+label, Toast.LENGTH_SHORT).show();
            alarmOff();
            checkDaysTotal(weeks);
            Log.d(TAG, "실행");
            finish();
        }
        else if(label.equals("BackPressed")){
            Toast.makeText(this, "Label 출석체크 취소", Toast.LENGTH_SHORT).show();
            mediaRestart();
        }
        else {
            Toast.makeText(this, "출석체크 실패 : "+label, Toast.LENGTH_SHORT).show();
            count++;
            mediaRestart();
            Log.d("count_number", ""+count);

            if(count>=3){ // count 가 3일 때 (사물인식 출석체크 3번 실패 시) count = 0으로 셋팅후 textRecognition 메소드 실행(text 인식 출석체크 Activity 실행)
                count = 0;
                Log.d("count_reset", ""+count);
                textRecognition();
            }
        }
        break;
    case TEXT_ACTIVITY :
        boolean checkValue = data.getBooleanExtra("checkValue", false);
        if(checkValue == true){
            Toast.makeText(this, "Text 출석체크 완료", Toast.LENGTH_SHORT).show();
            alarmOff();
            checkDaysTotal(weeks);
            finish();
        } else {
            Toast.makeText(this, "Text 출석체크 취소", Toast.LENGTH_SHORT).show();
            mediaRestart();
        }
        break;
    case IMAGE_MATCHING_ACTIVITY :
        boolean checkMatching = data.getBooleanExtra("checkMatching", false);
        if(checkMatching == true){
            Toast.makeText(this, "ImageMatching 출석체크 완료", Toast.LENGTH_SHORT).show();
            alarmOff();
            checkDaysTotal(weeks);
            finish();
        }else {
            Toast.makeText(this, "ImageMatching 출석체크 취소", Toast.LENGTH_SHORT).show();
            mediaRestart();
        }
    }
}
~~~
Skip button을 클릭시 수행되는 `dialogSkip()` method이다.<br>
AlertDialog를 띄워 출석 여부를 고를 수 있다.
~~~java
public void dialogSkip(){
    activity = this;
    AlertDialog.Builder alertdialog = new AlertDialog.Builder(activity);
    alertdialog.setMessage("출석여부를 고르세요.");

    // 확인버튼 - 결석
    alertdialog.setPositiveButton("결석", new DialogInterface.OnClickListener(){
        @Override
        public void onClick(DialogInterface dialog, int which) {
	    Toast.makeText(activity, "결석처리 되었습니다.", Toast.LENGTH_SHORT).show();
	    alarmOff();
	    finish();
        }
    });
    // 취소버튼
    alertdialog.setNegativeButton("출석", new DialogInterface.OnClickListener() {
        @Override
        public void onClick(DialogInterface dialog, int which) {
	    Toast.makeText(activity, "출석처리 되었습니다.", Toast.LENGTH_SHORT).show();
	    alarmOff();
	    checkDaysTotal(weeks);
	    finish();
        }
    });
    alertdialog.setNeutralButton("취소", new DialogInterface.OnClickListener(){
        @Override
        public void onClick(DialogInterface dialog, int id)
        {
	    Toast.makeText(activity, "'취소'버튼을 누르셨습니다.", Toast.LENGTH_SHORT).show();
	    mediaRestart();
        }
    });
    AlertDialog alert = alertdialog.create();
    alert.setTitle("Skip");
    alert.show();
}
~~~
>[AttendanceCheckActivity.java 전체 코드](https://github.com/JJinTae/MakeYouStudy/blob/master/app/src/main/java/com/android/MakeYouStudy/AttendanceCheckActivity.java)

## Firebase ML Kit를 이용한 출석체크 

Firebase ML Kit를 이용한 출석체크 기능으로는 **사물인식 (Image Labeling)**, **Text 인식 (Text Recognition)** 이 있다. 
- Firebase ML Kit를 사용하기 위해서는 아래와 같이 `build.gradle`에 정의해주어야 한다.

**build.gradle (:app)**
~~~java
implementation 'com.google.firebase:firebase-core:17.4.2'  
implementation 'com.google.firebase:firebase-ml-vision:24.0.0'  
~~~



**InternetCheck.java**<br>
인터넷 여부를 체크하기 위해 인터넷을 체크하는 Activity를 추가한다.
~~~java
public class InternetCheck extends AsyncTask<Void,Void,Boolean> {

    Consumer consumer;

    public InternetCheck(Consumer consumer){
        this.consumer = consumer;
        execute();
    }

    @Override
    protected Boolean doInBackground(Void... voids) {
        try{
            Socket socket = new Socket();
            socket.connect(new InetSocketAddress("google.com",80),1500);
            socket.close();
            return true;
        }catch (Exception e){
            return false;
        }


    }

    @Override
    protected void onPostExecute(Boolean aBoolean) {
        super.onPostExecute(aBoolean);
        consumer.accept(aBoolean);
    }

    public interface Consumer {
        void accept(boolean internet);
    }
}
~~~
***
### 사물 인식 (Image Labeling)
- 카메라로 책상을 촬영하여, 책상이 인식되면 출석체크가 완료된다.
- 사물인식 (Image Labeling) 기능을 사용할 때, 보다 안정적이고 빠르게 촬영 후 Detect하기 위해 CameraKit를 사용하였다.
- 촬영한 사진의 Label 값을 AttendanceCheckActivity로 전달한다.

>사물 인식을 통한 출석체크가 수행되는 과정
>1. **AttendanceCheckActivity**에서 사물 인식 기능 button을 클릭하여 **ImageLabelActivity**로 이동한다.
>2. **ImageLabelActivity**에서 cameraView를 통해 책상을 촬영한다.
>3. 촬영된 image에서 설정한 ConfidenceThreshold 값 이상인 Label중 가장 높은 Confidence를 가진 Label을 반환한다.
>4. **ImageLabelActivity**가 종료되면서 반환된 Label을 **AttendanceCheckActivity**로 넘겨준다.
>5. **AttendanceCheckActivity**에서 Label 값이 책상이 맞는지 확인한다. 
 
 Firebase ML Kit의 Image Labeling를 사용하기 위해 아래와 같이 `build.gradle`에 정의해주어야 한다.

**build.gradle (:app)**
~~~java
implementation 'com.google.firebase:firebase-ml-vision-image-label-model:19.0.0' 
~~~
>[Firebase MK Kit Image Labeling 설명 바로가기](https://firebase.google.com/docs/ml-kit/android/label-images)

CameraKit를 사용하기 위해 아래와 같이 `build.gradle`에  정의해주어야 한다.

**build.gradle (:app)**
~~~java
implementation 'com.wonderkiln:camerakit:0.13.1'
~~~
>[CameraKit github 바로가기](https://github.com/CameraKit/camerakit-android)


**ImageLabelActivity.java**
- ImageLabelActivity.java는 cameraKit의 cameraView와 Detect를 수행하는 Button을 사용한다.

>activity_image_label.xml
cameraKit의 cameraView를 사용하기 위하여 아래와 같이 layout에 추가한다.
>~~~java
><com.wonderkiln.camerakit.CameraView  
>  android:id="@+id/camera_view"  
>  android:layout_width="match_parent"  
>  android:layout_height="match_parent"  
>  android:layout_above="@+id/btn_detect">
></com.wonderkiln.camerakit.CameraView>
>~~~
아래 코드는 Detect button의 `onClickListener`이다. button을 클릭하면 cameraView가 실행되고 촬영된다.
~~~java
btnDetect.setOnClickListener(new View.OnClickListener(){
    @Override
    public void onClick(View v) {
	cameraView.start();
	cameraView.captureImage();
    }
});
~~~

`CameraKitListener()` 부분이다. cameraView에 바로 camera를 띄워 촬영한다.
~~~java
cameraView.addCameraKitListener(new CameraKitEventListener() {
    @Override
    public void onEvent(CameraKitEvent cameraKitEvent) { }

    @Override
    public void onError(CameraKitError cameraKitError) { }

    @Override
    public void onImage(CameraKitImage cameraKitImage) {
	waitingDialog.show();
	Bitmap bitmap = cameraKitImage.getBitmap();
	bitmap = Bitmap.createScaledBitmap(bitmap,cameraView.getWidth(),cameraView.getHeight(), false);
	cameraView.stop();

	runDetector(bitmap);
    }

    @Override
    public void onVideo(CameraKitVideo cameraKitVideo) { }
});
~~~
위의 `CameraKitListener()`에서 사용한 `runDetector()` method이다.<br>
아까 만들어준 InternetCheck.java 를 통해 인터넷을 체크한 후 , image에서 사물 인식 confidenceThreshold를 설정해준다.<br>
설정한 confidenceThreshold의 값보다 높은 값을 가지는 Label이 반환한다.
~~~java
private void runDetector(Bitmap bitmap) {
    final FirebaseVisionImage image = FirebaseVisionImage.fromBitmap(bitmap);

    new InternetCheck(new InternetCheck.Consumer() {
        @Override
        public void accept(boolean internet) {
	    if(internet)
	    {
		//인터넷이 있을 때 클라우드 사용
		FirebaseVisionCloudImageLabelerOptions options =
			new FirebaseVisionCloudImageLabelerOptions.Builder()
				.setConfidenceThreshold(0.7f) // 감지된 Label의 신뢰도 설정. 이 값보다 높은 신뢰도의 label만 반환됨
				.build();
		FirebaseVisionImageLabeler detector =
			FirebaseVision.getInstance().getCloudImageLabeler(options);

		detector.processImage(image)
				    .addOnSuccessListener(new OnSuccessListener<List<FirebaseVisionImageLabel>>() {
					@Override
					public void onSuccess(List<FirebaseVisionImageLabel> firebaseVisionCloudLabels) {
					    processDataResultCloud(firebaseVisionCloudLabels);
					}
				    })
				    .addOnFailureListener(new OnFailureListener() {
					@Override
					public void onFailure(@NonNull Exception e) { Log.d("EDMTERROR", e.getMessage()); }
		});

	    }
	    else
	    {
	        Toast.makeText(ImageLabelActivity.this, "인터넷을 체크하고 다시 촬영해주세요.", Toast.LENGTH_LONG).show();
	        ...
	    }
        }
    });
}

~~~
위의 `runDetector()` 에서 사용한 `processDataResultCloud()` method 이다.<br>
`runDetector()`에서 인식한 Label을 넘겨받아 Label 값이 존재할 경우 AttendanceCheckActivity로 Label 값을 넘겨주면서 ImageLabelActivity를 종료한다.
~~~java
private void processDataResultCloud(List<FirebaseVisionImageLabel> firebaseVisionCloudLabels) {
    if(firebaseVisionCloudLabels.size()!=0){
        for(FirebaseVisionImageLabel label : firebaseVisionCloudLabels)
        {
            String labeling = label.getText();

            Intent intent = new Intent();
            intent.putExtra("labeling", labeling);

            setResult(RESULT_OK, intent);
            finish();
        }
    }
    else{
        Intent intent = new Intent();
        intent.putExtra("labeling", "NULL");

        setResult(RESULT_OK, intent);
        finish();
    }
    ...
}
~~~
>[ImageLabelActivity.java 전체 코드](https://github.com/JJinTae/MakeYouStudy/blob/master/app/src/main/java/com/android/MakeYouStudy/ImageLabelActivity.java)
***

### Text 인식 (Text Recognition)

- 제시된 영어단어를 노트에 따라 적고 촬영하여 두 개의 단어가 일치하면 출석체크가 완료된다.
- 출석체크 완료 시, true 값을 AttendanceCheckActivity에 전달한다.
- 출석체크 실패 시, 재촬영을 요구하는 text를 띄워준다.

>Text 인식을 통한 출석체크가 수행되는 과정
>1. **AttendanceCheckActivity**에서 하나의 영어단어를 랜덤으로 **TextRecognitionActivity**로 이동하면서 넘겨준다.
>2. **TextRecognitionActivity**에서 제시된 영어단어를 노트에 따라 적고 촬영한다.
>3. 촬영된 image에서 단어를 인식하고, 인식된 단어와 제시된 영어단어를 비교하여 같을 시에 true값을 반환한다.
>4. **TextRecognitionActivity**가 종료되면서 반환된 true 값을 **AttendanceCheckActivity**에 전달한다. 
	( 제시된 영어단어와 같지 않을 시에는 Activity가 종료되지 않으며, 재촬영을 요구하는 text를 띄워준다.)
>5. **AttendanceCheckActivity**에서 ture 값을 받았을 경우 출석체크가 완료된다.

AttendanceCheckActivity에서 전달받은 영어 단어를 textView에 표시해준다.
~~~java
Intent intent = getIntent();  
data = intent.getStringExtra("English");  
textView.setText("똑같이 작성해주세요 : "+ data + "\n");
~~~
사진 촬영 button의 `onClickListener`이다. 
~~~java
captureImageBtn.setOnClickListener(new View.OnClickListener() {  
    @Override  
    public void onClick(View v) {  
        dispatchTakePictureIntent();  
    }  
});
~~~
사진 촬영 button을 눌렀을 경우 실행되는 method이다. 카메라를 실행시켜준다.
~~~java
private void dispatchTakePictureIntent() {  
    Intent takePictureIntent = new Intent(MediaStore.ACTION_IMAGE_CAPTURE);  
    if (takePictureIntent.resolveActivity(getPackageManager()) != null) {  
        startActivityForResult(takePictureIntent, REQUEST_IMAGE_CAPTURE);  
    }  
}
~~~
카메라로 촬영한 후에 실행되는 method이다.<br>
image를 Bitmap으로 저장하고 imageView에 촬영된 사진을 보여준 후, `detectTextFromImage()` method를 실행한다.
~~~java
@Override  
protected void onActivityResult(int requestCode, int resultCode, Intent data) {  
    super.onActivityResult(requestCode, resultCode, data);  
    if (requestCode == REQUEST_IMAGE_CAPTURE && resultCode == RESULT_OK) {  
        Bundle extras = data.getExtras();  
        imageBitmap = (Bitmap) extras.get("data");  
        imageView.setImageBitmap(imageBitmap);  
        detectTextFromImage();  
    }  
}
~~~
>Firebase ML Kit에 대한 method 설명은 아래 링크를 참고하면서 보면 도움이 될 것이다. <br>
>[Firebase ML Kit Text Recognition 설명 바로가기](https://firebase.google.com/docs/ml-kit/android/recognize-text)

위의 카메라로 촬영한 후에 실행되는 method에서의 `detectTextFromImage()` method이다.<br>
인터넷이 연결되어 있을 때, 촬영된 image에서 Text를 인식하고 성공했을 시에  [`FirebaseVisionText`](https://firebase.google.com/docs/reference/android/com/google/firebase/ml/vision/text/FirebaseVisionText) 객체가 성공 리스너에 전달된다.<br>
 
`displayTextFromImage()` method에 `FirebaseVisionText` 객체를 파라미터로 전달하여 실행한다.
> `FirebaseVisionText` 객체는 이미지에서 인식된 전체 텍스트 및 0개 이상의 [`TextBlock`](https://firebase.google.com/docs/reference/android/com/google/firebase/ml/vision/text/FirebaseVisionText.TextBlock) 객체를 포함한다.
~~~java
private void detectTextFromImage()
{
    new InternetCheck(new InternetCheck.Consumer() {
        @Override
        public void accept(boolean internet) {
            if(internet){
                FirebaseVisionImage firebaseVisionImage = FirebaseVisionImage.fromBitmap(imageBitmap);
                FirebaseVisionTextRecognizer firebaseVisionTextDetector = FirebaseVision.getInstance().getOnDeviceTextRecognizer();
                firebaseVisionTextDetector.processImage(firebaseVisionImage).addOnSuccessListener(new OnSuccessListener<FirebaseVisionText>() {
                    @Override
                    public void onSuccess(FirebaseVisionText firebaseVisionText) {
                        displayTextFromImage(firebaseVisionText);
                    }
                }).addOnFailureListener(new OnFailureListener() {
                    @Override
                    public void onFailure(@NonNull Exception e) {
                        Toast.makeText(TextRecognitionActivity.this, "Error: "+ e.getMessage(), Toast.LENGTH_SHORT).show();
                    }
                });
            }else {
                Toast.makeText(TextRecognitionActivity.this, "인터넷을 체크하고 다시 촬영해주세요.", Toast.LENGTH_LONG).show();
            }
        }
    });
}
~~~
`TextBlock` 를 List에 넣어주고, List의 size가 0일 때는 image에서 Text가 인식되지 않은 것이기 때문에 textView에 재촬영을 요구하는 글을 표시한다.<br>
Text가 인식된 경우에는 Text와 AttendanceCheckActivity에서 전달받은 단어를 `check()` method의 파라미터로 전달하여 실행한다. 
~~~java
private void displayTextFromImage(FirebaseVisionText firebaseVisionText) {
    List<FirebaseVisionText.TextBlock> blockList = firebaseVisionText.getTextBlocks();
    if(blockList.size() == 0){
        textView2.setText("사진에서 단어가 인식되지 않았습니다. 다시 촬영해주세요.");
    }
    else {
        String text = "";
        for(FirebaseVisionText.TextBlock block : firebaseVisionText.getTextBlocks())
        {
            text = block.getText().toLowerCase();
            check(text, data);
        }
    }
}
~~~
제시된 단어와 촬영하여 인식된 단어가 같은지 확인하는 `check()` method이다.<br>
두 단어가 일치할 시, 현재 Activity가 종료되면서 **AttendanceCheckActivity**로 true값을 전달한다.
~~~java
public void check(String text, String data){  
    if(text.equals(data)){  
        checkValue = true;  
	Intent intent = new Intent();  
	intent.putExtra("checkValue", checkValue);  
	setResult(RESULT_OK, intent);  
	finish();  
    }  
    else {  
        checkValue = false;  
	textView2.setText("인식된 단어는 " + text);  
    }  
}
~~~
>[TextRecognitionActivity.java 전체 코드](https://github.com/JJinTae/MakeYouStudy/blob/master/app/src/main/java/com/android/MakeYouStudy/TextRecognitionActivity.java)

## OpenCV를 이용한 출석체크

OpenCV를 이용한 출석체크를 하기위해서는, 미리 등록된 5장의 책상 사진이 존재해야한다.<br>
Profile에서 5장의 책상사진을 업로드하면 이 기능을 사용할 수 있다.
- OpenCV의 Color Histogram을 이용하여 두 장의 Image를 비교하는 기능을 구현하였다. 
- Color Histogram은 조명에 영향을 받을 수 있기 때문에 여러 장의 사진을 등록하여 비교한다.
- 등록한 사진과 같은 책상 사진을 찍었지만 출석체크가 되지 않은 경우에는 빛에 의해 사진의 밝기가 변화가 생겼기 때문일 수있다.
- 이러한 문제를 개선하기 위해, 출석체크에 실패했을 경우 해당 사진을 등록하는 책상 사진으로 추가할 수 있다.

>[OpenCV Histogram Compare 설명](https://docs.opencv.org/master/d8/dc8/tutorial_histogram_comparison.html)

>Color Histogram를 통한 출석체크가 수행되는 과정
>1. Profile (설정)에서 5장의 책상 사진을 미리 등록한다.
>2. 등록해놓은 책상 사진과 최대한 같은 각도로 책상을 촬영한다.
>3. 등록돼있는 사진들과 출석체크를 위해 촬영한 사진과 비교하여 일치율을 설정해놓은 Threshold와 비교하여 한 장이라도 만족할 시, **AttendanceCheckActivity**로  true 값을 전달한다.
>4. 등록한 사진 5장 모두 만족하지 않았을 경우, 촬영한 해당 사진을 등록할 것인지 Dialog를 통해 선택할 수 있다.    

### Color Histogram Image Matching 출석체크

**ImageMatchingActivity.java**

Camera를 실행시켜주는 button `onClickListener`이다.
~~~java
btnCamera = (Button)findViewById(R.id.btnCamera);  
btnCamera.setOnClickListener(new View.OnClickListener() {  
    @Override  
    public void onClick(View v) {    
    Intent intent = new Intent(MediaStore.ACTION_IMAGE_CAPTURE);  
    startActivityForResult(intent, 0);  
    }  
});  

~~~
Camera로 촬영한 후에 실행되는 method이다. 일치 여부를 나타내는 boolean변수를 false로, 비교를 성공한 image의 개수를 나타내는 count 값을 0으로 초기화해주고 `imageDownload()` method를 실행한다.
~~~java
@Override
protected void onActivityResult(int requestCode, int resultCode, @Nullable Intent data) {
    super.onActivityResult(requestCode, resultCode, data);
    if (requestCode == 0 && resultCode == RESULT_OK) {
        Bundle extras = data.getExtras();
        capturebmp = (Bitmap) extras.get("data");
        CheckSuccess = false;
        count = 0;
            
        imageDownload();
    }
}
~~~
등록되어 있는 image를 불러와서 방금 촬영한 image와 비교하는 `matching()` method를 실행한다.<br>
등록되어 있는 image는 bitmap 변수에 저장하고 방금촬영한 image는 `matching()`에 파라미터로 전달한다.<br>
한 장의 사진과 비교할 때마다 count값이 증가하며 총 5 장의 사진을 다 비교하고 true값을 반환하였다면, **ImageMatchingActivity**를 종료하고 true 값을 **AttendanceCheckActivity**로 전달한다.
~~~java
public void imageDownload(){
    storageRef.listAll().addOnSuccessListener(new OnSuccessListener<ListResult>() {
        @Override
        public void onSuccess(ListResult listResult) {
            for (StorageReference item : listResult.getItems()) {
            
                final long ONE_MEGABYTE = 1024 * 1024;
                item.getBytes(ONE_MEGABYTE).addOnSuccessListener(new OnSuccessListener<byte[]>() {
                    @Override
                    public void onSuccess(byte[] bytes) {
	                ...
                        bitmap = BitmapFactory.decodeByteArray(bytes, 0, bytes.length);

                        matching(capturebmp);
                        count++;

                        if(count== 5){
                            if(CheckSuccess == true){
	                        ...
                                Intent intent = new Intent();
                                intent.putExtra("checkMatching", CheckSuccess);
                                setResult(RESULT_OK, intent);
                                finish();
                            }else {
                                ...
                                dialogUpload();
                            }
                        }

                    }
                }).addOnFailureListener(new OnFailureListener() {
                    @Override
                    public void onFailure(@NonNull Exception exception) { }
                });
            }
        }
    })
    .addOnFailureListener(new OnFailureListener() {
        @Override
        public void onFailure(@NonNull Exception e) { }
    });
}
~~~
>[Firebase Storage File Download 사용법](https://firebase.google.com/docs/storage/android/download-files)
>[Firebase Storage File List 가져오기](https://firebase.google.com/docs/storage/android/list-files)

등록된 사진과 촬영한 사진을 비교하는 method이다. 촬영한 사진을 파라미터로 받아와서 등록되어 있는 사진과 비교한다.<br>
이 mathod를 위의 `imageDownload()` 에서 총 5번 수행하여 5장의 사진 모두 비교한다. <br>
각각의 이미지를 bitmap에서 Mat으로 변환을 해주고, [`Imgproc.cvtColor()`](https://docs.opencv.org/master/d8/d01/group__imgproc__color__conversions.html#ga397ae87e1288a81d2363b61574eb8cab)를 통해 HSV로 변환한다.<br>
[`Imgproc.calcHist()`](https://docs.opencv.org/master/d6/dc7/group__imgproc__hist.html#ga4b2b5fd75503ff9e6844cc4dcdaed35d)를 통해 color Histogram을 계산한 후, [`Core.normalize()`](https://docs.opencv.org/master/dc/d84/group__core__basic.html#ga1b6a396a456c8b6c6e4afd8591560d80)로 정규화해준다.<br>
각각 정규화까지 끝난 image를 [`Imgproc.compareHist()`](https://docs.opencv.org/master/d6/dc7/group__imgproc__hist.html#gaf4190090efa5c47cb367cf97a9a519bd)로 color Histogram을 비교하여 일치율을 `metric_val`변수에 넣어준다.<br>
metric_val값이 0에 가까울수록 일치율이 높은 결과이다.<br>
같은 책상사진을 촬영하였을 때, 다른 책상 혹은 다른 곳을 촬영하였을 때 등 여러 테스트를 거쳐 0.2값보다 작은 경우를 일치하는 것으로 판단하도록 구현하였다.<br>
0.2값보다 작은 image가 하나라도 존재한다면 true 값을 반환하여 출석체크가 가능하다.
~~~java
public void matching( Bitmap bitmap2){

    if(!OpenCVLoader.initDebug()){
        Log.d("start error : ", "OpenCV not loaded");
    } else {
        Log.d("start : ", "OpenCV loaded");
        try {
            Mat hist_1 = new Mat();
            Mat hist_2 = new Mat();

            MatOfFloat ranges = new MatOfFloat(0f, 256f);
            MatOfInt histSize = new MatOfInt(25);
	    
	    //등록된 사진
            img1 = new Mat();
            Utils.bitmapToMat(bitmap, img1);
            Imgproc.cvtColor(img1, img1, COLOR_BGR2HSV);
            Imgproc.calcHist(Arrays.asList(img1), new MatOfInt(0), new Mat(), hist_1, histSize, ranges);
            Core.normalize(hist_1, hist_1, 0, 1, Core.NORM_MINMAX);
	    
	    //촬영한 사진
            img2 = new Mat();
            Utils.bitmapToMat(bitmap2, img2);
            Imgproc.cvtColor(img2, img2, COLOR_BGR2HSV);
            Imgproc.calcHist(Arrays.asList(img2), new MatOfInt(0), new Mat(), hist_2, histSize, ranges);
            Core.normalize(hist_2, hist_2, 0, 1, Core.NORM_MINMAX);
	    
	    //두 사진 비교 후 결과
            metric_val = Imgproc.compareHist(hist_1, hist_2, Imgproc.HISTCMP_BHATTACHARYYA);// 0이 일치
            if(metric_val < 0.2) {
                CheckSuccess = true;
            }
            
        } catch (Exception e) { }
    }
}
~~~
출석체크에 실패했을 시 띄워주는 dialog이다. 
- 등록 버튼을 누를 시, 가장 오래된 사진을 하나 삭제하고 촬영한 해당 사진을 등록한다.
- 취소 버튼을 누를 시, 등록을 하지않고 다시 출석체크를 진행해야 한다.

여기서 사용되는 [`checksize()`](https://github.com/JJinTae/MakeYouStudy/blob/c3e8c9d4b3280c0fae93a51494ed29fe4fca873c/app/src/main/java/com/android/MakeYouStudy/ImageMatchingActivity.java#L285)와 [`imageUpload()`](https://github.com/JJinTae/MakeYouStudy/blob/c3e8c9d4b3280c0fae93a51494ed29fe4fca873c/app/src/main/java/com/android/MakeYouStudy/ImageMatchingActivity.java#L256)는 ProfileActivity에 있는 method와 동일하여 Link로 남겨두었다.
~~~java
public void dialogUpload(){
    activity = this;

    AlertDialog.Builder alertdialog = new AlertDialog.Builder(activity);
    alertdialog.setMessage("해당 사진을 등록하시겠습니까?");

    // 등록 버튼
    alertdialog.setPositiveButton("등록", new DialogInterface.OnClickListener(){
        @Override
        public void onClick(DialogInterface dialog, int which) {
            checksize();
            imageUpload(capturebmp);
        }
    });

    // 취소 버튼
    alertdialog.setNegativeButton("취소", new DialogInterface.OnClickListener() {

        @Override
        public void onClick(DialogInterface dialog, int which) {
            Toast.makeText(activity, "취소 되었습니다.", Toast.LENGTH_SHORT).show();
        }
    });


    AlertDialog alert = alertdialog.create();
    alert.setTitle("출석체크 실패");

    alert.show();

}
~~~
>[ImageMatchingActivity.java 전체 코드](https://github.com/JJinTae/MakeYouStudy/blob/c3e8c9d4b3280c0fae93a51494ed29fe4fca873c/app/src/main/java/com/android/MakeYouStudy/ImageMatchingActivity.java)
