# git의 3개의 tree

git은 파일의 변경사항을 관리하기 위해 크게 3가지 tree를 사용한다.

- working directory
- index
- HEAD

가 그것들이다.

### 첫번째 tree: working directory

- working directory는 쉽게 말해 개발자가 코드 에디터/탐색기 등을 통해 볼 수 있는 디렉토리이다.
- 개발자는 working directory내의 코드들을 수정하거나, 새로운 파일들을 추가 / 삭제 한다.

### 두번째 tree: index

- index는 staging area와 같은 말이다.
- commit시 index에 기록된 변경사항들이 HEAD가 가리키는 tree에 반영된다.
  - 즉, working directory에서 아무리 파일을 수정/추가/삭제 하였을 지라도, 변경사항들이 staging area에 올라가지 않으면, 아무리 commit해도 local repository에는 반영되지 않는다.

### 세번째 tree: HEAD

- HEAD는 현재 branch의 가장 최신 commit을 point한다.

## 응용

### git reset

### git commit
