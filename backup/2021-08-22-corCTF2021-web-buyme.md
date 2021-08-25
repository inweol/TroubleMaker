---
layout: post
date: 2021-08-23 00:00:00
title: "corCTF 2021 web/buyme writeup"
tags: [CTF]
---
## (Web) web/buyme [444 pts]

> 이 문제는 flag를 구매하는 함수의 로직이 잘못된 부분을 이용하여 해결하는 문제입니다.

```js
const fs = require("fs");

const flags = new Map();
for(let flag of JSON.parse(fs.readFileSync("flags.json")).flags) {
    if(flag.name === "corCTF") {
        flag.text = process.env.FLAG || "corctf{test_flag}";
    }
    flags.set(flag.name, flag);
}

const users = new Map();

const buyFlag = ({ flag, user }) => {
    if(!flags.has(flag)) {
        throw new Error("Unknown flag");
    }
    if(user.money < flags.get(flag).price) {
        throw new Error("Not enough money");
    }

    user.money -= flags.get(flag).price;
    user.flags.push(flag);
    users.set(user.user, user);
};

module.exports = { flags, users, buyFlag };
```
위 코드를 보면 Map() 클래스를 사용하여 users라는 객체를 생성합니다.

그리고 `buyFlag()`함수를 보면 `has()` 메소드를 사용해 `flags`라는 객체에 인자로 넘어온 `flag`가 존재하는지 확인합니다. 만약 `flag`가 존재한다면 인자로 넘어온 `user`객체의 `money`값이 `flags`의 값과 비교하여 적으면 Error를 반환하고 많으면 `flag`의 가격만큼 돈을 깎고 `user.flag`에 `flag`를 push하고 `users`객체를 덮어쓰는걸 확인할 수 있습니다.

정상적인 로직이라면 사용자가 가진 돈을 사용자의 입력값에서 가져오는 것이 아니라 `db`를 참조해서 가져와야 하는데, `buyflag()` 함수에서는 사용자가 입력한 데이터를 참조하기 때문에 `user`데이터를 조작해서 보낸다면 flag를 획득할 수 있습니다.

![image](https://user-images.githubusercontent.com/82700218/130357942-b38052f8-62d6-4d5d-8709-bcee020ac669.png)

회원가입 후 로그인을 하면 현재 가진 돈과 플래그가 나타납니다.

![image](https://user-images.githubusercontent.com/82700218/130357998-7588bb6c-6660-49c7-98d9-d0d88c265663.png)

`Flags`의 카테고리로 이동하면 구매할 수 있는 플래그들이 존재합니다. 구매해야할 플래그는 `corCTF`인데 가진 금액이 모자라 서 정상적으로는 구매가 불가능합니다.

```js
router.post("/buy", requiresLogin, async (req, res) => {
    if(!req.body.flag) {
        return res.redirect("/flags?error=" + encodeURIComponent("Missing flag to buy"));
    }

    try {
        db.buyFlag({ user: req.user, ...req.body });
    }
    catch(err) {
        return res.redirect("/flags?error=" + encodeURIComponent(err.message));
    }

    res.redirect("/?message=" + encodeURIComponent("Flag bought successfully"));
});
```
`/buy` 라우터를 보면 `buyFlag()`함수를 호출할 때 첫 번째 인자로 `req.user`의 값을 그대로 넘겨주는 것을 확인할 수 있고,  두 번째 인자로 Spread syntax를 이용해 값을 넘겨주고 있습니다.

`buyflag()`함수를 호출하기 전에 `req.user`에 대한 검증 절차가 존재하지 않아 자유롭게 조작이 가능합니다. 그래서 위에 분석한 `buyFlag()`함수의 잘못된 로직을 이용해 `json`으로 조작된 `user`의 데이터를 전달하여 플래그 구매를 시도 하였습니다.

**※**`json`으로 데이터를 전달해도 잘 동작하는 이유는 `app.use(express.json())`가 설정되어 있기 때문입니다.

![image](https://user-images.githubusercontent.com/82700218/130358810-219a8b7b-dc07-49be-99fa-4c06f26b16f5.png)

`response`를 통해 플래그 구매가 정상적으로 된것을 확인할 수 있습니다.

![image](https://user-images.githubusercontent.com/82700218/130358918-2cf88f10-3c81-4da8-b1f5-df0f0799105e.png)

그리고 다시 `home`으로 돌아가보면 `flag`를 획득할 수 있습니다.

> FLAG : `corctf{h0w_did_u_steal_my_flags_you_flag_h0arder??!!}`

---
