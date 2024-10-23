# Intro.
- **Title:** **Toward Formally Verifying Congestion Control Behavior**
- **Writers:** Venkat Arun , Mina Tahmasbi Arashloo , Ahmed Saeed, Mohammad Alizadeh, Hari Balakrishnan
- **Year:** 2021, presented during ACM SIGCOMM 2021 Conference (SIGCOMM ’21), August 23– 27
- **DOI:** [Toward formally verifying congestion control behavior | Proceedings of the 2021 ACM SIGCOMM 2021 Conference](https://dl.acm.org/doi/10.1145/3452296.3472912)
- For author's notes, see [here]([Formally Verifying Congestion Control Behavior (mit.edu)](https://projects.csail.mit.edu/ccac/))
- For implementation, see [here](https://github.com/venkatarun95/ccac/)

The main topic here is the design of internet Congestion Control Algorithms (CCAs).

The paper describes the Congestion Control Anxiety Controller, or CCAC (pronounced seek-ack).

>[!TLDR] 
> CCAC uses formal verification to establish certain properties of CCAs. 
> 
> It is able to prove hypotheses about CCAs or generate counterexamples for invalid hypotheses. With CCAC, a designer can not only gain greater confidence prior to deployment to avoid unpleasant surprises, but can also use the counterexamples to iteratively improve their algorithm.

Here, greater emphasis is on AIMD protocols, [[Copa]] and [[BBR]].

What CCAC hopes to achieve are:
- Allow for a formal verification of a CCA, to provide greater confidence in the operation of the CCA (i.e. relieve the *anxiety* that the designer might have about the algorithm, hence the name).
- Allow for simpler A/B tests of the protocol. 
  Usually one needs access to large CDNs to really test the protocol, but such test can be costly or unavailable to most designers.



