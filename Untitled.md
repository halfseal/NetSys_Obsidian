epoll, select() 이 두게 찾아보기, select epoll은 wait 안함
select dp file descriptor 넣어준다. 그거 확인하는 동안 event 확인해본다.
이게 폴링의 형식으로 진행된다. sock이 file descriptor, fopen 할때 리턴되는 거임
ㅅ시간마다 fd를 찾아보는 과정이 있다. 
read 까지 불리는 경로 찾기
event poll, 경로 찾아보기 socket 과 경로 좀 다를ㄹ것
user space 으로 올려주는 경로 찾아보기aa
tx sending 쪽으로 보기
compact 하게 진행하기
