https://www.kernel.org/doc/html/latest/filesystems/ext4/journal.html 

1. System Crash 발생 시, **파일 시스템** 에 대한 메타데이터의 불일치를 방지하기 위해 저널을 사용한다.
2. 최대 10,240,00개의 블록이 파일 시스템 내부에 저널을 위한 공간으로 예약될 수 있다.
3. 성능상이 이유로 ext4는 default 설정으로는 파일 시스템에 메타데이터만 Journal을 통해 기록한다.
  - 디스크 마운트시 data=ordered (default)
  - data=journal (모든데이터, 메타데이터가 저널을 통해 기록), 
  - data=writeback 사용시 dirty blcok은 메타데이터가 저널을 통해 디스크에 기록되기전에 디스크로 플러시 되지 않음 

약어 jdb2 =  Journaling Block Device version 2 
