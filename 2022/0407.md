# Today I Learn (0325)

## 1. 오늘의 프로젝트 구현 내용

### Trouble Shooting

### 이모티콘 사용시 User Socket id 유실이슈

**1. 문제점**

> 격려하기 기능 사용 시 사용한 사람의 화면에서는 이모티콘이 보이지만, 상대방의 화면에서는 보이지 않는 현상.
![이모티콘 사용시](https://user-images.githubusercontent.com/98517680/161411969-5931fc8c-8217-4ac7-9696-a53bbb0c2a65.png)

<이모티콘 사용예시>

**2. 문제 파악**

**2-1. 1차 파악**

초기에 생각했을 때, 자신의 화면에 나타나는 부분은 jquery의 영역이기 때문에, 문제는 socket에서 발생한다고 생각했다.

아래 코드는 프로젝트 socket을 통해 이모티콘을 주고 받는 부분이다.

```javascript
//자신의 화면에 이모티콘을 띄우는 부분
function showEmoji() {
  const myArea = document.querySelector("#mystream");
  const emojiBox = document.createElement("h1");
  emojiBox.innerText = "👍";
  myArea.appendChild(emojiBox);
  setTimeout(() => {
    emojiBox.hidden = true;
  }, 2000);
  socket.emit("emoji"); //socket 통신 부분.
}
```

위 코드에서 확인 할 수 있듯이, 이모티콘을 사용하는 사람이 socket을 통해 server에 "emoji"라는 이벤트를 보내면,

```javascript
socket.on("emoji", () => {
  socket.to(myRoomName).emit("emoji", socket.id);
});
```

server는 이벤트를 감지하여 방을 만들 때 생성 되었던 user의 socket id를 보내주게 되고

```javascript
// 여긴 다른 사람들에게 띄우는 부분
socket.on("emoji", (remoteSocketId) => {
  const remoteDiv = document.querySelector(`#${remoteSocketId}`);
  const emojiBox = document.createElement("button");
  emojiBox.innerText = "👍";
  remoteDiv.appendChild(emojiBox);
  setTimeout(() => {
    emojiBox.hidden = true;
  }, 2000);
});
```

방안에 있는 사람들은 remoteSocketId라는 이름으로 socketid를 받게 된다.<br> 그 후 동일한 socket id를 가진 div를 찾게 되는데, 이는 본인을 제외한 다른 사용자의 stream을 div에 넣을 때 해당 div의 아이디를 socket id로 하였기 때문이다. <br>코드는 아래와 같다.

```javascript
async function paintPeerFace(peerStream, id, remoteNickname) {
  try {
    const videoGrid = document.querySelector("#video-grid");
    const video = document.createElement("video");
    const peername = document.createElement("h3");
    const div = document.createElement("div");
    div.id = id; //다른 사용자들의 div아이디 = socket id
    video.autoplay = true;
    video.playsInline = true;
    video.srcObject = peerStream;
    peername.innerText = `${remoteNickname}`;
    peername.style.color = "white";
    div.appendChild(peername);
    div.appendChild(video);
    video.className = "memberVideo";
    peername.className = "nickNameBox";
    videoGrid.appendChild(div);
  } catch (error) {
    console.log(error);
  }
}
```

문제는 여기서 발 생하는 듯했다.

**즉, server에서 보내주는 socket id와 동일한 socket id를 가지고 있는 div를 찾지 못한다는 것이다.**

이러한 현상의 발생원인을 알기 위해서는, 제일 먼저 server에서 저장하는 socket id를 확인해 볼 필요가 있었다.<br> 왜냐하면 프론트 쪽에서는 보내주는 것 없이 받아서 사용하는데, 찾지 못한다는 건 뭔가 다른 socket id를 전송해주고 있다는 것을 의미하기 때문이다.

**2-2. 2차 파악**

**도대체 어떤 socket id를 받아 오길래, 맞는 div를 찾지 못하는걸까?**

아래 코드는 처음 방에 입장하게 되면 일어나는 내용이다.<br> (아래 코드가 실행 되기 전 방의 여부를 체크하는 logic이 있다.)

```javascript
socket.on("join_room", (roomName, nickname) => {
  //방 상태를 체크하는 로직이 진행 된 후, 방으로 입장하는 단계
  console.log("처음 방에 입장했을 때 생성되는 socket id", socket.id);
  socket.join(roomName);
  socket.emit("accept_join", targetRoomObj.users, socket.id);
});
```

아래 코드는 이모티콘을 눌렀을 때 socket id에 발생하는 변화를 확인하기 위한 내용이다.

```javascript
socket.on("emoji", () => {
  console.log("이모티콘을 눌렀을 때 새롭게 생성되는 socket id", socket.id);
  socket.to(myRoomName).emit("emoji", socket.id);
});
```

server단에서 확인한 두 개의 console.log에 담긴 내용를 비교해 보면,

![클릭할때마다 달라지는 socket id](https://user-images.githubusercontent.com/98517680/161412054-37648871-89f5-4ab2-8284-881055223bb1.png)

처음 2명의 유저의 입장으로 생성된 socket id와 이모티콘 event가 발생했을 때 생성되는 socket id가 다른 것을 찾을 수 있었다.

이러한 이유로 프론트에서 socket id를 받아도 맞는 div를 찾지 못했기 떄문에 해당 사용자에 이모티콘을 띄워줄수 없었던 것이다.

**2-3. 3차 파악**

**그렇다면 이렇게 다른 socket id가 생기는 이유는 뭘까**

> **socket 내부에서 생성되는 여러개의 sids때문일까?**
가장 먼저 socket이라는게 어떻게 생겨먹은 건지 궁금해서 console.log를 통해 확인하다 우연히 발견한 내용이라서, 이것이 확실한 원인이라고 할 수 있을지는 모르겠다.

socket을 콘솔에 찍어보면 굉장히 많은 내용이 나오게 되는데, 그중에서도 sids라는게 있었다.

![socket 콘솔로그](https://user-images.githubusercontent.com/98517680/161412124-3fdececa-2b79-4508-9704-02ac8acca9fa.png)

확인을 해보니 처음 입장하게 되었을 때 생성되는 socket id가 그안에 있었다.

![sids 안에 있는 socket id](https://user-images.githubusercontent.com/98517680/161412015-745a9faa-da06-487b-abbf-ad9c49372364.png)

그레서 이를 보고는 socket안에서 생성되는 여러개의 socket id중에 하나를 랜덤으로 배정하는 방식인가 했다.

그렇다고 하기엔 이모티콘 기능을 실행 했을 때 생성되는 socket id는 sids안에서 찾을 수 없었다.

그래서 이것이 원인은 아닌 듯 했다.

그래서 일단은 socket id의 불변성을 유지하기 위해한 방법을 찾아봐야 했다.

**3. 해결방안**

> **어떻게 socket id의 불변성을 유지할 수 있을까.**
사실 왜 socket id가 계속해서 변하는지에 대한 근본적인 원인을 찾지 못한 상태이지만, 일단 MVP 완성을 위해서 선택한 방법은 "useState"이다.

useState는 리액트 16.8 에서 도입된 기능 중 하나로 함수형 컴포넌트에서도 상태를 관리할 수 있게 하는 Hook이다.

먼저 사용자가 socket을 통해 입장하게 되면 server쪽에서 socket id를 보내주게 되는데,

```javascript
socket.join(roomName);
//socket id를 보내주는 부분
socket.emit("accept_join", targetRoomObj.users, socket.id);
socket.emit("checkCurStatus", mediaStatus[roomName]);
```

프론트에서는 그 아이디를 받아 useState로 관리하는 것이다.

```javascript
const [socketID, setSocketID] = useState("");
//
//
//
socket.on("accept_join", async (userObjArr, socketIdformserver) => {
  const length = userObjArr.length;
  //카메라, 마이크 가져오기
  await getMedia();
  setSocketID(socketIdformserver);
  changeNumberOfUsers(`${peopleInRoom} / 5`);
});
```

이렇게 저장된 socket id는 "emoji" event를 socket을 통해 server로 보낼 때 방이름과 같이 보냄으로서 불변성을 유지 할 수 있었다.

```javascript
// 내 화면에 띄우는 부분
   showEmoji: () => {
      const myArea = document.querySelector("#mystream");
      const emojiBox = document.createElement("img");
      emojiBox.src = HiFive;
      myArea.appendChild(emojiBox);
      setTimeout(() => {
        myArea.removeChild(emojiBox);
      }, 2000);
      emojiBox.className = "emojiBox";
      socket.emit("emoji", roomName, socketID);
    },
  }));
```

```javascript
// 여긴 다른 사람들에게 띄우는 부분
socket.on("emoji", (remoteSocketId) => {
  const remoteDiv = document.getElementById(`${remoteSocketId}`);
  const emojiBox = document.createElement("img");
  emojiBox.src = HiFive;
  emojiBox.className = "emojiBox";
  if (remoteDiv) {
    remoteDiv.appendChild(emojiBox);
    setTimeout(() => {
      remoteDiv.removeChild(emojiBox);
    }, 2000);
  }
});
```

**+ 추가 사항**

_useEffect로 묶이지 않은 socket Event 때문이라고?😨_

사실 이 부분은 멘토링 과정에서 코드 리뷰를 하다가 나온 내용이다. 멘토님이 코드를 보시고는 useEffect에 넣지 않으면 state가 바뀔 때 마다 페이지가 리-렌더링 될 것이고, 그러면 그때 마다 새로운 socket이 연결되는 것이라고 하셨다.

**즉, useEffect안에 socket Event들이 있지 않을 경우**

> state 변동 ➡ page 리렌더링 ➡ socket 재연결 ➡ socket id 재생성
그도 그럴 것이, 버튼들의 state를 제일 부모 컴포넌트에서 관리하고 있엇고, 영상통화을 관리하는 컴포넌트는 자식 컴포넌트 였기 때문에, 재렌더링으로 인해 socket이 계속해서 재연결되고 있었던 것이다.

그래서 이후 useState로 관리를 해주고 useEffect 안에 모든 socket함수를 넣고 나니, 렌더링 되는 과정도 브라우저에서 경량화 된 것을 체감할 수 있었다.

4. 결론

**Socket Event를 관리 할 때는 useEffect안에 넣어둠으로써 재렌더링을 방지하고, 데이터가 유실 될 경우 useState를 통해 관리할 수 있다.**
