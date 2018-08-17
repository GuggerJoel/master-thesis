# Competing Token Standards

The \glspl{eip} repository is open to everyone and anyone is free to suggest any \gls{eip}. Many people correctly identified the drawbacks of ERC20 as explained in section \ref{strength-and-weaknesses-of-erc20} and many amendments to ERC20 have been proposed. Those amendments are problematic as they change the established standard, migrating to a newer and improved token standard is a better solution---which is the goal behind ERC777. Moreover ERC777 is not the only or even the first new token standard to be proposed to replace ERC20. It is also is not the last, as ERC777 gain popularity a fe more related standard and other token standards started to appear on the \glspl{eip} repository \ref{eipsrepo}.

In this chapter, we will explore three of the main tokens standard proposals competing with ERC777. The first one is ERC223 which predates ERC777 and looked very promising and gained some community support as it was for a time the only real alternative to ERC20. The second one came after ERC777 as indicated by its number: ERC8275. In the same way ER223 tries to be an answer to the drawbacks of ERC20, ERC777 and ERC827 try to be an answer to the drawbacks of ERC20 and the issues from ERC223.

## ERC223

ERC223 was submitted on March 5^th^ 2017, by a developer knows as Dexaran. It has one clear goal in mind: to address the issue of accidentally locking token in ERC20 (see section \ref{locked-tokens}).

The solution suggested by this proposal is to define a `tokenFallback` function similar to the default fallback function \citepalias[see][Fallback Function]{soldoc}. This function takes as parameters the address of the spender (`from`), the amount of tokens transferred and a `data` field. Any contract wishing to receive tokens must implement this function.

The ERC223 proposal requires the token contract to check whether the recipient is a contract when sending tokens and if it is, then the token contract must call the `tokenFallback` function on the recipient directly.

Overall ERC223 does bring some improvements, first it provides a simplified `transfer` function which does away with the `approve` and `transferFrom` mechanism of ERC20. Secondly it adds a `data` field to attach information to a token transfer. The whole standard is quite short and kept simple which may be an advantage as it makes it easy for potential developers to understand and thus may reduce mistakes when implementing tokens.

Although, the proposal may also be consider to simplistic and limited, thus creating token which are too limited and forcing developers to add non standard functionalities to provide the behaviors they need. This exact scenario is already happening with ERC20.

Moreover, the standard defines two functions named `transfer` with a different number of parameters (one with and one without the `data` parameter). This overloading is allowed in Solidity, however other high level languages such as Vyper---a new language compiling to \gls{evm} bytecode and inspired by the Python syntax---which is slowing gaining traction in the community may not support function overloading. Hence token contracts following ERC223 can never be implemented in Vyper which is a sever limitation to the growth of both ERC223 and newer languages such as Vyper.

Furthermore, because recipient contract are required to implement a specific function, which virtually almost no deployed contracts implements today. As a result, adoption of ERC223 implies that all existing contracts are migrated to new updated version if they wish to support ERC223 tokens which is another even more severe limitation to the growth and adoption of ERC223. to try and alleviate this restriction, the repository of the reference implementation for ERC223 contains an implementation which allows custom callbacks. In other words, it lets the spender specify which function to call on the recipient. While this may appear as an improvement, it actually opens the significant security breach as ERC827 which potentially allows hijacking of the contract (see section \ref{erc827} for details).

The proposal also has some inaccurate claims such as backward compatibility with ERC20. The author appears to confuse backward compatibility with the ERC20 standard and ERC20 compatible functional interfaces.

> Now ERC23 is 100% backwards compatible with ERC20 and will work with every old contract designed to work with ERC20 tokens. \flushright (Dexaran, comment on ERC223)

Specifically, both standard define an identical `transfer` function as part of their interface. Therefore, some contract capable of calling the ERC20 `transfer` function will be capable of calling the exact same transfer function on an ERC223 token contract. Nevertheless this does not imply compatibility between the two standards. The behavior of the `transfer` function changes widely form one standard to the next and this change of behavior may break things. Potentially a contract could handle transferring ERC20 tokens by first checking if the recipient is a contract or not and call `transfer` or `approve` accordingly. If this contract is given an ERC2223 token, it may try to call the `approve` function on the ERC223 token which does not implement the function and the transaction will fail.

> ERC777 has been built to solve some of the shortcomings of ERC223. Please have a look at it:
>
> \url{http://eips.ethereum.org/EIPS/eip-777} \flushright (chencho777, comment on ERC223)

Finally the developer behind the standard appears to be more focus on solving the issue of locked tokens despite the concerns mentioned above and raised by the community. Ultimately there was a feeling that an agreement would be hard to reach, community members became more and more doubtful regarding the viability of ERC223 and the standard started to become more and more stagnant, with the last comments suggesting to look at ERC777 instead.

> \@MicahZoltu is 100% correct. This discussion did not lead to a consensus, so don't expect this standard to be followed. [...] \flushright (Griff Green, comment on ERC223)


## ERC827

ERC827 is another proposal to fix ERC20. Unlike ERC777 which takes a more independent approach which is fully dissociated form ERC20 and where both standard can be implemented side-by-side, the ERC827 proposal tries to build a second standard on top of ERC20.

This proposal is overall rather short as well. It defines three additional functions to reduce the changes of locking tokens---`transferAndCall`, `approveAndCall` and `transferFromAndCall`. Fundamentally, those three functions are wrapper around the ERC20 `transfer`, `approve` and `transferFrom` functions where the wrapper function calls another function--whose name and parameters are initially passed to the wrapper function.

This approach is simple and does provide full backward-compatibility with the ERC20 standard. Nevertheless, it does suffer from some significant hindrance. First, because this standard simply builds on top of ERC20, all the drawbacks of ERC20 remain present, including the issues related to `decimals` and the possibility to lock tokens.

Secondly, passing both the name and the data (i.e the parameters) of the function to call in the `transferAndCall`, `approveAndCall` and `transferFromAndCall` functions implies that there is no guaranteed way to communicate directly to that function the actual amount of tokens being transferred. Some token contract may for example automatically levy a transfer fee in tokens, or the token may represent some currency with demurrage and part of the amount is burned when transferring. In other words, to know the actual amount transferred, a recipient should keep track if its balance internally, call the `balanceOf` function and from there it can compute the amount received and update the internal balance. This is both tedious and expensive in gas to do. And there sis always the risk of the state of the internal balance diverging from the balance in the contract, for example calling the ERC20 `transfer` function will increase the balance in the token contract but not in the recipient contract.

Finally, the contract suffers from a significant security flaw. Essentially the three functions added by ERC827 allow anyone to perform arbitrary call from the token contract which is a security risk \citep{consensysrecommendations} and in this context the same security flaw as the implementation of ERC223 with custom fallback---which is essentially the same mechanism of allowing spenders to execute custom calls via the token contract.

In greater details, the flaw was exploited live in the ATN token \citep{atnreport} \citep{secbit2018lacking}, an instance of the ERC223 implementation containing the flawed custom fallback. The attack comes from the unsafe assumption that a spender will pass a function to call on the recipient such that the recipient is able to react to the delivery of tokens. Albeit this may be the intended use it cannot be enforced and the spender is free to specify any function that the token contract will then call. For the ATN token contract\footnote{\href{https://etherscan.io/address/0x461733c17b0755ca5649b6db08b3e213fcf22546}{Deployed at 0x461733c17b0755ca5649b6db08b3e213fcf22546}}, the attacker decided to transfer zero tokens (a transfer of `0` token is considered valid) to the token contract itself. Therefore the token contract was also the recipient contract and it will call any function on itself. This is an interesting scenario as access control in Ethereum is often enforced by looking at the address from which the call originated (`msg.sender` in Solidity). Often some functions are only executed if they are called by the owner of the contract (the address which deployed the contract in the first place) or the contract itself. The `ds-auth` library applies this principle exactly---as shown in listing \ref{lst:dsauth}---and it was taken advantage of by the attacker.

```{caption="The \texttt{auth} modifier and the \texttt{setOwner} function from the \texttt{ds-auth} library which let the ATN attacker gain ownership of the token contract." label="lst:dsauth" language=solidity}
function setOwner(address owner_)
    public
    auth
{
    owner = owner_;
    emit LogSetOwner(owner);
}

// ...

modifier auth {
    require(isAuthorized(msg.sender, msg.sig));
    _;
}

function isAuthorized(address src, bytes4 sig) internal view returns (bool) {
    if (src == address(this)) {
        return true;
    } else if (src == owner) {
        return true;
    } else if (authority == DSAuthority(0)) {
        return false;
    } else {
        return authority.canCall(src, this, sig);
    }
}
```


When the attacker issued the transaction\footnote{\url{https://etherscan.io/tx/0x3b7bd618c49e693c92b2d6bfb3a5adeae498d9d170c15fcc79dd374166d28b7b}} to transfer zero tokens it then asked the token contract to call the `setOwner` function on itself. Normally the access to this function is limited thanks to the use of `ds-auth` but in this case the call came from the contract itself and was therefore authorized. Obviously the attacker set his own address as the new owner and thus gain control of the contract. At this point the attacks has the ability to call functions---as the owner---that other addresses are not able to call, namely the `mint` function to issue new tokens. With its second transaction\footnote{\url{https://etherscan.io/tx/0x9b559ffae76d4b75d2f21bd643d44d1b96ee013c79918511e3127664f8f7a910}} the attacked mined eleven million ATN tokens for itself. Finally, in an attempt to cover its tracks, the attacker set the owner back to the original owner with a third transaction\footnote{\url{https://etherscan.io/tx/0xfd5c2180f002539cd636132f1baae0e318d8f1162fb62fb5e3493788a034545a}}.

>TODO add diagram of the attack