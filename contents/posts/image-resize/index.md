---
title: "서버에서 이미지 리사이징하기"
description: "자바스크립트와 쉘 스크립트를 구성해봅시다."
date: 2023-10-03
update: 2023-10-03
tags:
  - javascript
  - 이미지
  - webp
---

> 이 글은 우테코 피움팀 크루 '[주노](https://github.com/Choi-JJunho), [참새](https://github.com/WaiNaat)'가 작성했습니다.

## 서론

피움 서비스를 이용하는 과정에서 불러오는 이미지의 크기가 너무 커서 사용자가 이미지 로드할 때 시간이 오래 걸린다.

> ![](.index_images/79ba487e.png)
> 위와 같은 환경에서 이미지 로드 시간을 책정해봤다.
> ![](.index_images/5020cb2e.png) 
> 사용자가 인터넷 환경이 느린 경우 1.8MB 사진을 로드하는 데 9.99초 이상의 로드시간이 걸린다.

피움 서비스는 사진 업로드가 유의미할 정도로 많이 일어나지 않는다.

따라서 다음과 같은 시나리오로 이미지를 압축하려고한다.

```markdown
1. S3에서 이미지 불러오기
2. 이미지 압축
3. S3로 이미지 보내기
```

서버에서 특정 시간마다 이미지 압축을 수행하고 이를 S3에 업로드하는 과정을 통해 이미지 최적화를 진행해보자.

## 시작하기

> WebP(웹피)는 손실/비손실 압축 이미지 파일을 위한 이미지 포맷이다.
> [위키백과](https://ko.wikipedia.org/wiki/WebP)

이미지 압축을 진행하기에 앞서 다음 두가지 방식을 고려했다.

- `1. 서버에서 cwebp 프로그램을 이용해 이미지를 webp 형태로 변환하는 방법`
- `2. node, python과 같은 언어를 이용해 이미지를 변환하는 방법`

1번의 경우 서버에 직접 프로그램을 설치하고 관리해야하는 관리 소요가 있다

현재 빌드서버에서 배치로 작업을 수행할 것이기 때문에 npm이 이미 설치되어있다는 가정하에 javascript로 개발하는 것이 적합하다고 판단하여 node.js로 이미지 최적화 코드를 작성헀다.

추가로 해당 코드를 주기적으로 실행시키기 위한 쉘 스크립트도 함께 작성했다.

## 코드

코드를 하나씩 살펴보자.

### 자바스크립트

내부적으로 Sharp 라이브러리를 사용하여 변환하고 .png, .webp 형태로 제공된다.

### 쉘 스크립트

파일을 탐색하여 자바스크립트로 작성된 함수를 수행한다.

```shell
# 전처리

convert_image() {
    local count_success=0
    local count_failure=0
    local count_skip=0

    local target_dir=$1
    local size=$2
    
    local total_files=$(find "$target_dir" -type f -iname "*.jpeg" -o -iname "*.jpg" -o -iname "*.png" -o -iname "*.gif" | wc -l)

    echo "$total_files 개의 파일에 대한 작업 시작"

    # 대, 소문자 구분 없이 탐색 활성화
    shopt -s nocaseglob
    for file in "$target_dir"/*.{jpeg,jpg,png,gif}; do
	    # 대, 소문자 구분 없이 탐색 비활성화
	    shopt -u nocaseglob

	    if [ -f "$file" ]; then # 파일인 경우에만 처리
		    filename_full=$(basename "$file")
		    extension="${filename##*.}" # 확장자 추출
		    extension=$(echo "$extension" | tr '[:upper:]' '[:lower:]')  # 확장자 소문자로 변환
		    filename="${filename_full%.*}"

		    output_file="$target_dir/${filename}.${nick_name}.webp"
		    if [ -e "$output_file" ]; then # 이미 존재하는 파일이라면 생략
			    ((count_skip+=1))
		    else
			    # 여기에 이미지 압축!
			    node resize.js -d "$target_dir" -i "$filename_full" -s "$size" -n "$nick_name"

			    if [ $? -ne 0 ]; then
				    echo "변환 실패: $file -> $output_file" >> "$error_log"
				    ((count_failure++))
			    else
				    ((count_success++))
			    fi
		    fi
	    fi
    done
    echo ""
    echo "SIZE $width 변환 실패 파일 개수: $count_failure"
    echo "SIZE $width 변환 성공 파일 개수: $count_success"
    echo "SIZE $width 변환 생략 파일 개수: $count_skip"
}

# 함수 실행
```

### crontab

아래 쉘 스크립트를 crontab에 등록하여 이미지 저장 - 압축 - 등록 과정을 수행한다.

```shell
#!/bin/bash

# S3에서 이미지를 받아온다
aws s3 sync s3://{S3 이미지 경로} {로컬 이미지 경로}

# 최적화를 진행한다.
/home/ubuntu/resize-image/bash.sh -s 512 -n "small" -d "{로컬 이미지 경로}"
/home/ubuntu/resize-image/bash.sh -s 128 -n "x-small" -d "{로컬 이미지 경로}"

# 최적화가 완료된 파일을 S3로 업로드한다.
aws s3 sync {로컬 이미지 경로} s3://{S3 이미지 경로}
```

## 정리

간단하게 배포 시나리오와 별개로 이미지 리사이징을 수행해봤다.

개발 일정이 이미 어느정도 잡혀있고 코드상으로 추가하기 어려운 부분이라면 서버에서 배치로 돌리는것도 고려해보면 좋을 것 같다.

- [github 이미지 리사이징을 위한 코드](https://github.com/pium-official/resize-image)

## Reference

- https://certbot.eff.org/instructions
