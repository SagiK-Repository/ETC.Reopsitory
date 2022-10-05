# MFC Picture Control 사용법

- 버튼을 누를 때 Picture Control에 Bitmap 띄우기
  ```cs
  void CTestDlg::OnBnClickedButton1()
  {
      CRect rect; //픽쳐 컨트롤의 크기를 저장할 CRect 객체
      CDC* dc; //픽쳐 컨트롤의 DC를 가져올  CDC 포인터
      dc = m_picture_control.GetDC(); //픽쳐 컨트롤의 DC를 얻는다.
      m_picture_control.GetWindowRect(rect);//GetWindowRect를 사용해서 픽쳐 컨트롤의 크기를 받는다.
  
      CImage image; //불러오고 싶은 이미지를 로드할 CImage 
      image.Load(_T("5.bmp")); //이미지 로드
      image.StretchBlt(dc->m_hDC,0,0,rect.Width(),rect.Height(),SRCCOPY);//이미지를 픽쳐 컨트롤 크기로 조정
   
      ReleaseDC(dc); //DC 해제
  }
  ```
- 참조 : https://garage-fullof-dummy.tistory.com/68

<br>

# 카페에서 제공하는 함수로 활용한 Picture Control 사용법

- OpenCV Mat를 Picture Control에 띄우기 (함수 사용)
  ```cs
  if (m_Image.data != NULL) {
      CRect rect; //픽쳐 컨트롤의 크기를 저장할 CRect 객체
      CDC* pDC; //픽쳐 컨트롤의 DC를 가져올 CDC 포인터
      pDC = m_View.GetDC(); //픽쳐 컨트롤의 DC를 얻는다.
      m_View.GetClientRect(rect); //GetWindowRect를 사용해서 픽쳐 컨트롤의 크기를 받는다.
    
      DisplayImage(pDC, rect, m_Image);  // 카메라에서 읽어들인 영상을 화면에 그리기
      
      ReleaseDC(pDC); //DC 해제
  }
  ```
- Bitmap을 가져오고 띄우는 부분을 DisplayImage()함수가 해주고 있는 것을 알 수 있다.
- Mat to Bitmap 작업을 진행한 다는 사실을 알 수 있다.
- 참조 : https://cafe.daum.net/smhan/darS/2

<br>

# 함수 분석

- 1차 대략 분석 (맥락을 분석한다)
  ```cs
  void DisplayImage( CDC* pDC, CRect rect, Mat& srcimg )
  {
      // 이미지 로딩 부분-----------------------------
      Mat img;
      int step = ((int)(rect.Width() / 4)) * 4;
      resize( srcimg, img, Size( step, rect.Height() ) );
      uchar buffer[sizeof( BITMAPINFOHEADER ) * 1024];
      //---------------------------------------------
      // 저장될 비트맵 이미지 공간
      BITMAPINFO* bmi = (BITMAPINFO*)buffer;
      
      // 이미지 속성 로딩-----------------------------
      int bmp_w = img.cols;
      int bmp_h = img.rows;
      int depth = img.depth();
      int channels = img.channels();
      int bpp = 8*channels;
      //---------------------------------------------
     
      // 이름으로 유추해 볼 때, 비트맵 공간 생성
      // Windows용 비트맵 정보를 생성합니다.
      FillBitmapInfo( bmi, bmp_w, bmp_h, bpp, 0 );
     
      // 출력 속성 지정 ------------------------------
      int from_x = MIN( 0, bmp_w - 1 );
      int from_y = MIN( 0, bmp_h - 1 );
      int sw = MAX( MIN( bmp_w - from_x, rect.Width() ), 0 );
      int sh = MAX( MIN( bmp_h - from_y, rect.Height() ), 0 );
      // --------------------------------------------
      
      // Picture Control에 출력----------------------
      SetDIBitsToDevice( pDC->m_hDC, rect.left, rect.top, sw, sh, from_x, from_y, 0, sh, img.data + from_y*img.step, bmi, 0 );
      img.release();
      //---------------------------------------------
  }
  ```
- 위 코드의 순서를 다시 명시하자면
  - 1. 이미지 정보 로딩
  - 2. 비트맵 저장공간 선언
  - 3. 이미지 속성 로딩
  - 4. 이미지 공간 확보
  - 5. 출력 속성 지정
  - 6. Mat와 비트맵 공간을 활용한 출력


### SetDIBitsToDevice()

- 비트맵 파일을 화면에 출력하는 함수입니다.
- Windows API는 DIB(장치 독립 비트맵)를 화면에 출력하기 위한 두 개의 함수를 지원한다.
  - 비트맵을 원본 크기 그대로 화면에 출력하는 SetDIBitsToDevice 함수
  - 비트맵을 원하는 크기로 확대 또는 축소시켜서 화면에 출력할 수 있는 StretchDIBits 함수
  ```cs
  int SetDIBitsToDevice(
      HDC hdc,                 // 출력 대상의 DC 핸들
      int XDest,               // 출력 대상의 좌상귀 x 좌표
      int YDest,               // 출력 대상의 좌상귀 y 좌표
      DWORD dwWidth,           // DIB 원본 사각형 너비(가로 픽셀 크기)
      DWORD dwHeight,          // DIB 원본 사각형 높이(세로 픽셀 크기)
      int XSrc,                // DIB 원본의 좌상귀 x 좌표
      int YSrc,                // DIB 원본의 좌상귀 y 좌표
      UINT uStartScan,         // 첫 번째 스캔 라인
      UINT cScanLines,         // 출력할 스캔 라인의 개수
      CONST VOID *lpvBits,     // 픽셀 데이터 시작 주소
      CONST BITMAPINFO *lpbmi, // BITMAPINFO 구조체 시작 주소
      UINT fuColorUse          // RGB 또는 색상 테이블 인덱스
  );
  ```
- 카페의 경우는 다음과 같다.
  ```cs
  SetDIBitsToDevice(
      pDC->m_hDC, // Picture Control의 DC에 있는 m_hDC
      rect.left, // 출력 대상의 시작 x위치 (Picture Control의 위치)
      rect.top,  //                 y위치
      sw,        // 원본 너비
      sh,        //      높이
      from_x,    // 원본 시작 x위치
      from_y,    //           y위치
      0,         // 첫 번째 스캔 라인 = 0
      sh,        // 출력할 스캔 라인의 개수
      img.data + from_y * img.step, // 픽셀 데이터 시작 주소
      bmi,       // 비트맵 구조체 시작 주소
      0          // RBG 또는 색상 테이블 인덱스
  );
  ```
- 여기서 img.data + from_y * img.step를 자세히 알아본다.
  - img.data : char로 이루어진 배열이다. (char[] 또는 Char*와 같다)
    - img.data는 Mat img에 존재하는 모든 자료를 한줄로 정렬한 것이다.
    - (x,y)에 있는 img.data를 조회하려면, img.data[(x + y * img.col)*img.step + n]을 진행하면 된다. 여기서 n은 차원수이다.
  - img.step : 행렬의 한 행이 차지하는 바이트 수




uchar* img_data = img.data; 


```cs
void DisplayImage( CDC* pDC, CRect rect, Mat& srcimg )
{
 Mat img;
 int step = ((int)(rect.Width() / 4)) * 4; // 4byte 단위조정해야 영상이 기울어지지 않는다.
 resize( srcimg, img, Size( step, rect.Height() ) );
 uchar buffer[sizeof( BITMAPINFOHEADER ) * 1024];
 BITMAPINFO* bmi = (BITMAPINFO*)buffer;

 int bmp_w = img.cols;
 int bmp_h = img.rows;
 int depth = img.depth();
 int channels = img.channels();
 int bpp = 8*channels;

 FillBitmapInfo( bmi, bmp_w, bmp_h, bpp, 0 );

 int from_x = MIN( 0, bmp_w - 1 );
 int from_y = MIN( 0, bmp_h - 1 );
 int sw = MAX( MIN( bmp_w - from_x, rect.Width() ), 0 );
 int sh = MAX( MIN( bmp_h - from_y, rect.Height() ), 0 );

 SetDIBitsToDevice( pDC->m_hDC, rect.left, rect.top, sw, sh, from_x, from_y, 0, sh, img.data + from_y*img.step, bmi, 0 );
 img.release();
} 
```

```cs
void FillBitmapInfo(BITMAPINFO* bmi, int width, int height, int bpp, int origin)
{
 assert(bmi&&width >= 0 && height >= 0 && (bpp == 8 || bpp == 24 || bpp == 32));
 BITMAPINFOHEADER *bmih = &(bmi->bmiHeader);
 memset(bmih, 0, sizeof(*bmih));
 bmih->biSize = sizeof(BITMAPINFOHEADER);
 bmih->biWidth = width;
 bmih->biHeight = origin ? abs(height) : -abs(height);
 bmih->biPlanes = 1;
 bmih->biBitCount = (unsigned short)bpp;
 bmih->biCompression = BI_RGB;
 if (bpp == 8) {
  RGBQUAD *palette = bmi->bmiColors;
  for (int i = 0; i<256; i++)  {
   palette[i].rgbBlue = palette[i].rgbGreen = palette[i].rgbRed = (BYTE)i;
   palette[i].rgbReserved = 0;
  }
 }
}
```
