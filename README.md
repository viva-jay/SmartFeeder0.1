## 프로젝트 설명

MAX-4466 센서를 이용하여 반려견의 소리가 일정이상 지속되면 등록된 스마트폰으로 PUSH하여 급식과 음악/동영상 재생이 가능하도록 하는 반려견 급식기 입니다.



## 개발 환경

* RaspberryPi3, Aduino

* Android 

* C,C++

  

## 구현기능

* mobile에서 넘어오는 급식,음악재생,동영상 재생 명령 처리와 반려견 정보 갱신 처리

* mobile에 소리 정보를 알려주기 위한 GCM PUSH서버 처리

  

## 주요 구현 코드

### Mobile에서 넘어온 데이터 처리

```c
void connect_mobile()
{
	while (1) 
	{
		int moblieClient_sock= accept(mobileServer_sock, (struct sockaddr *)&mobileClient_addr, &mclient_len);
		printf("Client is connected : %s\n", inet_ntoa(mobileClient_addr.sin_addr));
		int cnt =0, dataNum;
		char buf[16];
		char inputTemp[10][16], *token,tokenTemp[16]=".";
		while(1)
		{
			memset(buf,0x00,sizeof(buf));
			if((dataNum = read(moblieClient_sock, buf, 16)) <=0)
			{
				perror("read( )");
				break;
			}
			token = strtok(buf,tokenTemp);
			while(token!=NULL)
			{
				strcpy(inputTemp[cnt],token);
				token = strtok(NULL,tokenTemp);
				cnt++;
			}
			
		}
		
		//안드로이드로 부터 넘어온 요청에 따른 동작
		if(!strcmp(inputTemp[0],"motor"))
		{
			if(pthread_create(&id_motor,NULL,execute_motor,NULL) < 0)
             		{
        			perror("motor thread create error : ");
               		}
                        pthread_detach(id_motor);
			
		}elseif(!strcmp(inputTemp[0],"music"))
		{
			if(isMusicStart)
			{
				isMusicStart = 0;
				if(pthread_create(&id_hawlMusic,NULL,execute_HawlMusic,NULL) < 0)
				{
					perror("music thread create error : ");
				}
				pthread_detach(id_hawlMusic);
				}
				else
				{
					printf("Music is alreay start\n");
					system("kill $(pgrep omxplayer)");
				}
			}
		}
		elseif(!strcmp(inputTemp[0],"update"))//정보 갱신하기
		{
			strcpy(doggyInfo.age,inputTemp[0]);
			strcpy(doggyInfo.weight,inputTemp[1]);
			strcpy(doggyInfo.amount,inputTemp[2]);
		
		}
		close(moblieClient_sock);
	}	
```

### GCM PUSH 서버

```c
#include <arpa/inet.h>
#include <assert.h>
#include <errno.h>
#include <netinet/in.h>
#include <signal.h>
#include <stdlib.h>
#include <stdio.h>
#include <string.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <sys/wait.h>
#include <netdb.h>
#include <unistd.h>
 
#define MAXLINE 4096
#define MAXSUB  500

#define LISTENQ 1024
 
extern int h_errno;
 
const char *apikey = "";
const char *token = ""
 
ssize_t process_http(int sockfd, char *host, char *page, char *poststr)
{
    char sendline[MAXLINE + 1], recvline[MAXLINE + 1];
    ssize_t n;
    snprintf(sendline, MAXSUB,
            "POST %s HTTP/1.0\r\n"
            "Host: %s\r\n"
            "Content-type: application/json\r\n"
            "Authorization:key=%s\r\n"
            "Content-length: %d\r\n\r\n"
            "%s", page, host, apikey, strlen(poststr), poststr);
 
    printf("sendline : \n%s\n",sendline);
    write(sockfd, sendline, strlen(sendline));
    while ((n = read(sockfd, recvline, MAXLINE)) > 0)
    {
        recvline[n] = '\0';
        printf("%s", recvline);
    }
    return n;
 
}
 
int main(int argc, char** argv)
{
    int pushServer_sock;
    struct sockaddr_in pushServer_addr;
   
    char **pptr;
    char *hostname = "gcm-http.googleapis.com";
    char *page = "/gcm/send";
    char *poststr = "{ \"data\": {"
                           "\"title\": \"알림\","
                           "\"message\": \"개가 1분 이상 짖고 있습니다.\"},"
                           "\"to\" : \""
                            token
                            "\"}\r\n";
 
 
 
    char str[50];
    struct hostent *hptr;
    if ((hptr = gethostbyname(hostname)) == NULL) 
    {
        fprintf(stderr, " gethostbyname error for host: %s: %s",hostname, hstrerror(h_errno));
        exit(1);
    }
    printf("hostname: %s\n", hptr->hostname);
    if (hptr->h_addrtype == AF_INET && (pptr = hptr->h_addr_list) != NULL) 
    {
        printf("address: %s\n",inet_ntop(hptr->h_addrtype, *pptr, str,sizeof(str)));
    } 
    else
    {
        fprintf(stderr, "Error call inet_ntop \n");
    }
 
    pushServer_sock = socket(AF_INET, SOCK_STREAM, 0);
    bzero(&pushServer_addr, sizeof(pushServer_addr));
    pushServer_addr.sin_family = AF_INET;
    pushServer_addr.sin_port = htons(80);
    inet_pton(AF_INET, str, &pushServer_addr.sin_addr);
 
    printf("connection...\n");
    connect(pushServer_sock, (sockaddr*) & pushServer_addr, sizeof(pushServer_addr));
    printf("process http...\n---------------------------\n%s\n----------------------\n",poststr);
    process_http(pushServer_sock, hostname, page, poststr);
    close(pushServer_sock);
    return 0;
}
```

## 결과사진

![result1](https://user-images.githubusercontent.com/37570093/43032345-6266909a-8cef-11e8-932f-33537488dae1.PNG)
![result2](https://user-images.githubusercontent.com/37570093/43032354-784fc944-8cef-11e8-992b-75a9c62cc099.PNG)
