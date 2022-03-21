===============================================
YosysHQ SVA AXI Verification IP
===============================================

--------
Abstract
--------
TBD

------------
Organisation
------------
TBD

------------
Introduction
------------
Everyone in the industry has ever heard about Verification IP (VIP), how these magical things can save
hours of development, testing and integration. Somehow, CAD vendors sweeten the ears of all of us who work
hard trying to verify some complex component. They say, is just as easy to connect it and tada!, all the
bugs are shown so engineers can solve them quickly.

Then, you get the VIP and collateral (**$omehow**) and you realize that:
    * The documentation is not very understandable.
    * Barely has any quick start example, or the one that is delivered is as trivial as useful.
    * It contains a lot of scripts, setup configurations and all sort of things that are not compatible
      with the configuration of the design that you are testing right now. So you need to do more work to
      integrate a component for the component that you want to integrate into your project.
    * Among other things.

Where is then the Return of Investment (ROI) and low Time To Market (TTM) that the company that sells
the VIP was talking about?.

Despite of this bad first approach, you decide to go ahead and give it a chance, It can't be that bad, right?.
After pressing the <prove> button, the first counterexample (CEX) appears. By opening the wave to do root-cause analysis the first thing you see is this:

+----------------------------------------------------------------------+
| .. image:: img/encrypt.png                                           |
|    :width: 6.5in                                                     |
|    :height: 2.93in                                                   |
|    :align: center                                                    |
+======================================================================+
| Figure 1.1. A common CEX in a Formal VIP.                            |
+----------------------------------------------------------------------+

The assertion does not hold at clock cycle 5, and you cannot trace back *legal_wrap_pre* nor *legal_wrap_con*
signals because they are **encrypted**. Sometimes there is a message accompanying the property in the *fail statement* (where statements like *else*, *$info*, *$error*, etc, can be employed) that you can use to read the respective section of the specification where such behavior is described. For simple invariant (properties that holds despite input stimulus) such as handshakes, there is no complication. But for properties that depends on composed behaviors (data flow trackers like the CAM that most companies uses to track AXI IDs) is another story. if you cannot see what is going on inside, it is very easy to get lost: The AMBA spec says how thins should behave, but not how everyone should build a VIP. The VIP tells you have a problem, but it does not help you to debug and fix it.

But, to be fair, not everything is CAD companies fault. Developing a VIP is complicated. The engineers who write these VIPs needs to understand with great detail the specification, and cross words (emails) to the specification architects to clarify one thing or another. Speaking for example of the AMBA AXI and ACE Protocol Specification [missing link], almost everything is explained there, some things more detailed, some others miss  important information, or the second piece of info is indexed in any other part of the document or even worse, in another specification (try to read Atomic Accesses related sections knowing that any missing link is hidden in the Exclusive Monitor instructions (LDXR/STXR) programmers guide, because is ARM, and they share the same principles for most of their products).
