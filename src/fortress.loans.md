---
marp: true
theme: gaia
---

<style>
@import url('https://fonts.googleapis.com/css2?family=Montserrat:wght@300;400;500;600;700&display=swap');

:root {
    padding: 4rem;
    background-color: #fff;
    font-family: 'Montserrat', sans-serif;
}

p {
    font-size: .8rem;
}

h1 {
    color: #000;
}

h2 {
    color: #0a0a0a;
    font-size: 1.2rem;
}

li {
    font-size: .6rem;
}

a {
    font-weight: 300;
    text-decoration: none;
    color: #7393B3;
}
</style>

![w:150px](https://pbs.twimg.com/profile_images/1425178490786156545/rG-A6Bcc_400x400.jpg)

# Rekt: Fortress protocol

Fortress Protocol, кредитное подразделение JetFuel Finance на BSC 9 марта 2022 года было ограблено на $3M.

---

## Введение

Утром, 9 мая, Fortress Protocol подвергся хакерской атаке. Основными причинами атаки послужили изъян в governance-контракте и уязвимость оракула Umbrella Network.

---

## tl;dr || DYOR

Fortress -- это алгоритмический денежный рынок и протокол синтетических стейблкоинов, предназначенный для безопасного и надежного кредитования пользователей в Binance Smart Chain.

Адреса, задействованные в атаке, перечислены ниже:

- [Кошелек хакера BSC](https://bscscan.com/address/0xA6AF2872176320015f8ddB2ba013B38Cb35d22Ad)
- [Кошелек хакера ETH](https://etherscan.io/address/0xA6AF2872176320015f8ddB2ba013B38Cb35d22Ad)
- [Транзакция](https://bscscan.com/tx/0x13d19809b19ac512da6d110764caee75e2157ea62cb70937c8d9471afcb061bf)
- [Контракт FTS](https://bscscan.com/address/0x4437743ac02957068995c48e08465e0ee1769fbe)

---

## Подготовка к атаке

Подготовка к атаке началась [как минимум за 19 дней](https://etherscan.io/txs?a=0xA6AF2872176320015f8ddB2ba013B38Cb35d22Ad) до совершения.

29 апреля хакер [получил 20 ETH на TornadoCash](https://etherscan.io/tx/0x1f1b43b6a56698af777c8c8b7e70eb77f10ff08bd8518c1685b9c19528e3daa5) (протокол для осуществления частных транзакций в сети Ethereum, простыми словами миксер), и отправил 12.4 ETH на BSC, используя [Cbridge](https://cbridge.celer.network/#/transfer).

Стоит заметить что [выведены деньги были тем же способом.](https://etherscan.io/tx/0x36a0cdb1403ec2026b3878e23cec8904d142ef4b81c6fdeec0b694d7cb0c19c9)

---

## Миксер за работой

![w:1100px](https://sun1.userapi.com/sun1-19/s/v1/if2/v4nD-p0Mej6aKHqgO3XRWw-wK4A_IW9WvtnyOX45cF1GMSdkgt9xUsVxldiqTkUsnjNPbphyZghcZDQTU_j846gf.jpg?size=2560x727&quality=95&type=album)

---

## $FTS

[$FTS](https://bscscan.com/address/0x4437743ac02957068995c48e08465e0ee1769fbe) -- governance токен Fortress Protocol. Он был выпущен 21 апреля 2022 года с в количестве 10,000,000.

Благодаря низкой цене $FTS, хакер потратил всего 11.4 ETH на покупку 400,000 $FTS, что представляет из себя 4% всего supply.

---

## Вредоносный proposal

4 мая хакер задеплоил вредоносный proposal-контракт:
[BSCScan](https://bscscan.com/address/0x0dB3B68c482b04c49cD64728AD5D6d9a7B8E43e6)

Пропоузал был добавлен в очередь как Предложение 11.

Голосование за пропоузал проходило три дня, в это время атакующие голосовали за, используя часть приобретенных токенов $FTS. Голосование окончилось 7 Мая.

---

## Compound Governor

Fortress использует протокол [Compound Governor Alpha](https://github.com/compound-finance/compound-protocol/blob/master/contracts/Governance/GovernorAlpha.sol).

```js
function state(uint proposalId) public view returns (ProposalState) {
        // ... a bunch of proposal success conditions
        } else if (proposal.forVotes <= proposal.againstVotes || proposal.forVotes < quorumVotes()) {
            return ProposalState.Defeated;
        }
        ...
    }
```

```js
function quorumVotes() public pure returns (uint) { return 400000e18; } // 400,000 = 4% of FST
```

4% токенов -- одно из условий прохождения пропоузала.

---

## Вредоносный контракт

После окончания голосования, пропоузал проходит одобрение. Для вступления в силу нужно подождать два дня.

Спустя два дня, 9-го мая, пользователь создает еще один вредоносный контракт и официально запускает атаку.

Заключается она в изменении кредитного плеча $FTS с `0` до `0.7`. Это позволяет хакеру использовать 70% ценности приобритенных токенов для выпуска кредитов от лица протокола.

---

## Collateral price contract

Fortress Protocol использует [контракт для цен](https://bscscan.com/address/0x00fcF33BFa9e3fF791b2b819Ab2446861a318285), который обращается к разным ораклам для разных токенов

```js
function getPrice(address underlying) internal view returns (uint) {
        // ... if statements for other tokens
        } else if (underlying == FTS_ADDRESS) {
            return getUmbrellaPrice(ftsKey);
        else {
            return getChainlinkPrice(getFeed(underlying));
        }
    }
```

---

## Umbrella Oracle

Для получения цены $FTS Fortress обращается к [Umbrella Oracle](https://bscscan.com/address/0xc11B687cd6061A6516E23769E4657b6EfA25d78E)

```js
for (; i < _v.length; i++) {
      address signer = recoverSigner(affidavit, _v[i], _r[i], _s[i]);
      uint256 balance = stakingBank.balanceOf(signer);
      require(prevSigner < signer, "validator included more than once");
      prevSigner = signer;
      if (balance == 0) continue; // <---!!

      emit LogVoter(lastBlockId + 1, signer, balance);
      power += balance; // no need for safe math, if we overflow then we will not have enough power
    }
    require(i >= requiredSignatures, "not enough signatures"); // <---!!
```

---

## Уязвимость Umbrella

Пока количество подписей превышает значение `requiredSignatures`, проверка того, имеет ли подписавший право выставлять цену актива, не выполняется. Это позволило хакеру обойти вспе проверки и указать цену $FTS, которая выгодна

После атаки Umbrella Network исправила уязвимость и выпустила [обновленный контракт](https://bscscan.com/address/0x49D0D57cf6697b6a44050872CDb760945B710Aab).

---

## Новый контракт Umbrella Network

```js
for (uint256 i; i < _v.length; i++) {
      address signer = recoverSigner(affidavit, _v[i], _r[i], _s[i]);
      uint256 balance = stakingBank.balanceOf(signer);

      require(prevSigner < signer, "validator included more than once");
      prevSigner = signer;
      if (balance == 0) continue;

      signatures++; // <---!!

      emit LogVoter(lastBlockId + 1, signer, balance);
      power += balance; // no need for safe math, if we overflow then we will not have enough power
    }

    require(signatures >= requiredSignatures, "not enough signatures"); // <---!!
```

Метод `submit()` теперь проверяет разрешения сайнеров.

---

## Суть атаки

Вернемся к самой атаке. Злоумышленник воспользовался уязвимостью оракула и смог установить очень высокую цену $FTS, позволяя выпустить в долг множество активов, среди них -- BNB, USDC, USDT, BUSD, BTCB, ETH, LTC, XRP, ADA, DAI, DOT и SHIB.

Все эти токены были конвертированы примерно в три миллиона долларов США и отправлены в сеть Ethereum через Anyswap и cBridge. Затем USDT был конвертирован в ETH и DAI замиксован в TornadoCash.

---

## Схема движения средств

![w:900px](https://camo.githubusercontent.com/e971e4ebe88d03357643840d9d4b6b5efc1426d5ef73b4a5207c7c8909d8b3c9/68747470733a2f2f7062732e7477696d672e636f6d2f6d656469612f46535362343457586f4145725159743f666f726d61743d6a7067266e616d653d6c61726765)

---

## Выводы, `I`:

Хакер воспользовался уязвимостью в governance-протоколе и изменил цену актива в оракле.

Контракты Fortress Protocol были аудированы компаниями [Hash0x](https://fortress.loans/audit_hash0x.pdf) и [EtherAuthority](https://fortress.loans/audit_etherautherity.pdf). Уязвимостей ими не было обнаружено.

Эти компании являются относительно новыми на рынке. При осузествлении аудита стоит так же обращаться к лидерам рынка.

---

## Выводы, `II`:

Compound Governor является стандартом индустрии. Однако Fortress использовал более старую версию протокола (Alpha). Возможно, стоит присматриваться к более современным решениям, например, [Governor Bravo](https://compound.finance/docs/governance#governor-bravo).

При разработке governance-решений нужно уделять пристальное внимание логике голосования. Помимо того что это, очевидно, core-функционал, абузы governance приводят к очень серьезным последствиям.

---

Источники:

- https://slowmist.medium.com/slowmist-fortress-protocol-hack-analysis-19af24af723c
- https://rekt.news/fortress-rekt/
- https://coincodecap.com/fortress-protocol-hacked-around-3m-stolen
- https://cryptoadventure.com/fortress-protocol-breached-loses-3-million/
- https://www.stealthlabs.com/news/fortress-protocol-lost-usd-3-mn-in-oracle-price-manipulation-attack/

<br />
<br />

[semkin.eth](https://twitter.com/vasemkin), for [superdao](https://superdao.co/)
