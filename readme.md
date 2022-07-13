Drone18
====

대회진행전략
====

2022 미니드론 경진대회에서 우리 팀의 전략은 빠른시간 내에 불규칙적인 장애물을 통과하는 것이 목적이다. 이를 위해 matlab에서 제공하는 image processing toolbox를 사용하여  rgb2hsv, imcrop, regionprops 함수 등으로 이미지 처리를 수행하였다. 

전체적인 과정으로 우선 takeoff 이후 카메라 전방에 있는 천막 내 x,y 직경을 측정하여 구멍을 만든다. 다음으로는 만들어낸 구멍을 이용하여 화면상 특정한 위치에 존재하도록 드론을 제어한다. 이후에 가까워졌다고 인식하면 구멍 너머의 표식을 인식하여 통과하고 회전임무를 수행한다. 

코드를 구현하기 위한 공간을 마련하기 위해 필요한 장비(천막, 봉) 구매와 강의실을 대여하여 환경을 구축하였다. 주어진 작업환경에서 코딩 후 즉시 적용해가며 이상적인 값과 실제 적용한 값과의 오차를 줄여나갔다. 


알고리즘 설명
====
1. 기기의 객체 선언 및 takeoff 진행한다.
2. 파란색 천막을 인식 및 구멍을 이미지 처리한 후 드론 카메라에 표시되는 구멍의 위치에 따라 드론이 전진과 동시에 상하좌우로 제어한다.
3. 구멍에 충분히 가까워졌다고 판단되면 구멍을 이용한 제어는 멈추고 이후의 표식을 인식한다. 
4. 인식한 표식까지 전진한 후, 회전 임무를 수행한다. (2단계에서 2-3번을 반복진행)

# 소스코드 설명
1. 기기의 객체 선언 및 takeoff
<pre>
<code>
clc; clear;

%% 변수 선언
count = 0;
point = [480 360]; %카메라 각도 고려

red_h_min1 = 0; red_h_max1 = 0.05; red_h_min2 = 0.95; red_h_max2 = 1; red_s_min = 0.6; red_s_max = 1;
pur_h_min = 0.6; pur_h_max = 0.85; pur_s_min = 0.4; pur_s_max = 1;
gre_h_min = 0.3; gre_h_max = 0.4; gre_s_min = 0.4; gre_s_max = 1;
blu_h_min = 0.55; blu_h_max = 0.7; blu_s_min = 0.5; blu_s_max = 0.9;

%% 객체 선언  
drone = ryze(); %드론 객체 선언
cam = camera(drone);

%% Main 함수
takeoff(drone);
</code>
</pre>
-------------
2. 구멍 이미지 및 구멍 위치에 따라 일정한 시간간격으로 전진하는 드론을 상하좌우로 제어
<pre>
<code>
for mission = 1:3
    if mission == 1
        disp('미션 1 수행중');
    
    elseif mission == 2
        disp('미션 2 수행중');
        
    elseif mission == 3
        disp('미션 3 수행중');  
    end
 
    %% 원 통과 함수(Blue Screen Detection)
    while 1
        %이미지 처리(RGB->HSV)
        frame = snapshot(cam);
        hsv = rgb2hsv(frame);
        h = hsv(:,:,1);
        s = hsv(:,:,2);
        v = hsv(:,:,3);
        
        blue_screen = (blu_h_min<h)&(h<blu_h_max)&(blu_s_min<s)&(s<blu_s_max);
        circle = imfill(blue_screen,'holes');
         
        for x=1:size(blue_screen,1)
            for y=1:size(blue_screen,2)
                if blue_screen(x,y)==circle(x,y)
                    circle(x,y)=0;  %1:흰색, 0:검은색
                end
            end
        end
       
        %구멍이 식별된 경우
        if sum(circle,'all') > 10000 
            disp('구멍 탐색 완료!');
            break;

        %구멍이 식별되지 않을 경우
        else
            %화면의 좌우를 비교
            diff_lr = sum(imcrop(blue_screen,[0 0 480 720]),'all') - sum(imcrop(blue_screen,[480 0 480 720]),'all');
            diff_ud = sum(imcrop(blue_screen,[0 0 960 360]),'all') - sum(imcrop(blue_screen,[0 360 960 720]),'all');

            %장애물에 대한 이미지의 좌우 차이값이 30000이상이면 좌우로 이동
            if diff_lr > 30000
                moveleft(drone,'distance',0.3,'speed',1);
                disp('왼쪽으로 0.25m 만큼 이동');

            elseif diff_lr < -30000
                moveright(drone,'distance',0.3,'speed',1);
                disp('오른쪽으로 0.25m 만큼 이동');
            end

            %장애물에 대한 이미지의 상하 차이값이 10000이상이면 상하로 이동
            if diff_ud > 10000
                moveup(drone,'distance',0.2,'speed',1);
                disp('위쪽으로 0.2m 만큼 이동');
            elseif diff_ud < -10000
                movedown(drone,'distance',0.2,'speed',1);
                disp('아래쪽으로 0.2m 만큼 이동');
            end
        end
    end
    
    %% 원 통과 함수(Circle Detection)

    %% 원 통과 함수(전진 구동)
    while 1
        %이미지 처리(RGB->HSV)
        frame = snapshot(cam);
        hsv = rgb2hsv(frame);
        h = hsv(:,:,1);
        s = hsv(:,:,2);
        v = hsv(:,:,3);
        
        blue_screen = (blu_h_min<h)&(h<blu_h_max)&(blu_s_min<s)&(s<blu_s_max);
        fill_screen = imfill(blue_screen,'holes');
        circle = fill_screen;
        
        for x=1:size(blue_screen,1)
            for y=1:size(blue_screen,2)
                if blue_screen(x,y)==circle(x,y)
                    circle(x,y)=0;  %0:흰색, 1:검은색
                end
            end
        end
        
        detect_area = regionprops(circle,'Centroid','Area');
        area = 0;

        for j = 1:length(detect_area)
                if area <= detect_area(j).Area %가장 큰 영역 추출을 위하여 Area를 이용한 처리
                    area = detect_area(j).Area;
                    center = detect_area(j).Centroid;
                end
        end
        area
        imshow(circle);
        
        if area >= 80000
            break;
        end
                        
        if area ~= 0
            if (420 <= round(center(1)) && 540 >= round(center(1))) && (160 <= round(center(2)) && 240 >= round(center(2)))
                disp('1');
                    
                if area >= 50000
                    disp(['미션 ',mission, '원 통과 완료']);
                    break;
                    
                else
                    moveforward(drone, 'Distance', 0.6, 'speed', 1);
                end

            elseif 420 > round(center(1))
                moveleft(drone, 'Distance', 0.2, 'speed', 1);
                disp('2');

            elseif 540 < round(center(1))
                moveright(drone, 'Distance', 0.2, 'speed', 1);
                disp('3');

            elseif 160 > round(center(2))
                moveup(drone, 'Distance', 0.2, 'speed', 1);
                disp('4');

            elseif 240 < round(center(2))
                movedown(drone, 'Distance', 0.2, 'speed', 1);
                disp('5');
            end
            
        else
            disp(['미션 ',mission, '원2 통과 완료']);
            break;
        end
        
            
       
    end
</code>
</pre>