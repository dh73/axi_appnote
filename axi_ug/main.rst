===============================================
YosysHQ SVA AXI Verification IP
===============================================

--------
Abstract
--------
Due the popularity of AMBA protocols, AXI4 specifically, and trying to help our community to develop high quality designs with formal methods as easy as possible, we decided to create a Formal VIP for this protocol (yes, once again, but this time we tried to do it right). So in this AppNote we show you the results.

This AppNote will cover the following topics:
    * A quick glance to the newly and refreshed (SVA-AXI4-FVIP)[https://github.com/YosysHQ-GmbH/SVA-AXI4-FVIP].
    * Parameters and configuration descriptions.
    * An usage example in an interconnect design.


------------
Introduction
------------
Everyone in the industry has ever heard about Verification IP (VIP), how these magical things can save
hours of development, testing and integration. Somehow, CAD vendors sweeten the ears of all of us who, at
some point, needed to verify some complex component. They say, is just as easy to connect it and tada!, all the
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
| .. image:: ../img/encrypt.png                                        |
|    :width: 6.5in                                                     |
|    :height: 2.93in                                                   |
|    :align: center                                                    |
+======================================================================+
| Figure 1.1. A common CEX in a Formal VIP.                            |
+----------------------------------------------------------------------+

The assertion does not hold at clock cycle 5, and you cannot trace back *legal_wrap_pre* nor *legal_wrap_con*
signals because they are **encrypted**. Sometimes there is a message accompanying the property in the *fail statement* (where statements like *else*, *$info*, *$error*, etc, can be employed) that you can use to read the respective section of the specification where such behavior is described. For simple invariant (properties that holds despite input stimulus) such as handshakes, there is no complication. But for properties that depends on composed behaviors (data flow trackers like the CAM that most companies uses to track AXI IDs) is another story. if you cannot see what is going on inside, it is very easy to get lost: The AMBA spec says how thins should behave, but not how everyone should build a VIP. The VIP tells you have a problem, but it does not help you to debug and fix it.

But, to be fair, not everything is CAD companies fault. Developing a VIP is complicated. The engineers who write these VIPs needs to understand with great detail the specification, and cross words (emails) to the specification architects to clarify one thing or another. Speaking for example of the AMBA AXI and ACE Protocol Specification [missing link], almost everything is explained there, some things more detailed, some others miss  important information, or the second piece of info is indexed in any other part of the document or even worse, in another specification (try to read Atomic Accesses related sections knowing that any missing link is hidden in the Exclusive Monitor instructions (LDXR/STXR) programmers guide, because is ARM, and they share the same principles for most of their products).

----------------------------
AXI4 Formal VIP Architecture
----------------------------

Features
-----------------------------------------

The features and limitations of the new AXI4 SVA FVIP are the following ones:
    * Support for AXI4-Lite and AXI4-full.
    * Single transaction ID.
        * In-order transactions.
        * No interleaving.
    * No transaction caching, buffering or modifications.
        * Same bus width on all interfaces.
    * Written in a using simple SystemVerilog constructs for portability.
    * ARM recommended checks:
        * Low power channel interface.
        * X-propagation interface.
        * Deadlock checks.
    * A comprehensible documentation.
        * User guide.
        * Methodology guide.
        * Examples.
        * Instantiation templates.


.. note::
    Full AXI4 implementation is possible. In fact, at the moment of writing this AppNote, we have the capacity to test more than one transaction at a time, out-of-order transactions, full exclusive transaction monitors, data interleave, etc. But for simplicity purposes, and because these features covers most of the cases, we decided to release the IP in this state. The released FVIP has the required logic to add these features easily, that is another advantage of the open source components.

Architecture
-----------------------------------------

We designed the AXI4 SVA FVIP having in mind the fundamental architectural descriptions in the AMBA AXI4 IHI0022E spec (A1.3 AXI Architecture):
    * Each channel (W, AW, B, AR, R) is defined on its own module, and each module contains only the properties that are necessary for the AXI4 channel.
        * In this way, each verification engineer can focus on certain channel without the hurdle of loading tons of checks that are not of interest for the test in question.
        * Also, design engineers can incrementally add features or changes to an IP and get immediate feedback on the correctness of the implementation, again, without adding information that might not be required.
    * The properties are organized using SystemVerilog packages, and each package contains only the properties mentioned in the chapter of the spec.
        * This helps to disable checks that are not required, are proven, etc, as well as isolating properties for further investigation. And of course, to have a better understanding of what is required to implement the interfaces correctly.
        * We also include the *amba_axi4_protocol_checker.sv* which is a general *out-of-the-box protocol checker* with all channels instantiated and all properties enabled.
    * There is a separation between AMBA AXI rules and FVIP implementation libraries.
        * All explicit references in AMBA AXI4 IHI0022E are under `axi4_spec` directory.
        * All of the libraries and implementations that are not explicitly stated in the spec, are under `axi4_lib` directory.

    * A number of configuration knobs so the FVIP can be as flexible as possible.
        * One advantage of not having an encrypted IP is that the properties can be extended for cases like IPs that does not strictly follows the AMBA spec in some aspects, which is common in the industry.
    * Easy as possible debugging.
        * Each property has messages that points to the reference in the AMBA AXI4 IHI0022E, so upon failure, the user can just open the document, lookup for the page number and compare the design behavior to whatever is defined in the spec.
        * Some `let binders` are helpful to root-cause issues when calculations or temporal transactions are utilised. When they are deasserted, the user can follow the definition of the `let binder` and easily find the time where that requirement failed, and why.
        * Properties receive the signals of interest as arguments, so its easy to add them in the waveform (for tools that automatically opens debugger with COI signals, you will have everything you need in zero time).
    * And last but not least, the implemented checks are compliant with ARM AMBA AXI4 IHI0022E.
        * That means, we did not just defined things based in our interpretation of the descriptions in the spec, but followed them strictly.
        * We developed an infrastructure to verify our implementation based on information that is publicly available at ARM website.

.. note::
    We are an small company, we have no partnership with ARM at all so we could throw questions at each of the things that were not clear, so if there is any misinterpretation we will be happy to know, but at the moment, we have no seen any divergence between results of public ARM verification IP and ours.

The *Figure 2.1* shows the architecture of the AXI4 SVA FVIP. For more information refer to the *UG_verification_plan, Section 6 Architecture*.

+----------------------------------------------------------------------+
| .. image:: ../img/org.png                                            |
|    :width: 6.5in                                                     |
|    :height: 2.93in                                                   |
|    :align: center                                                    |
+======================================================================+
| Figure 2.1. Architecture and file organisation.                      |
+----------------------------------------------------------------------+


---------------------------------------------------
Formalisation and Optimisation of the AXI4 SVA FVIP
---------------------------------------------------

When to use BMC or K-induction
---------------------------------------------------

All of the properties defined in the IHI0022E spec are invariants, that is, they must hold *invariably* of the design input values and/or initial states. A good rule of thumb is to use *BMC* for the AXI control signals, such as handshakes, strobes, etc, and start with BMC but move incrementally to K-induction for data transport checks, such as properties for *channel relationships* or whenever tracking of "in-flight" data is needed. Although BMC with sufficient radius can be enough to gain confidence.

Bounded Model Checking (BMC) with AXI SVA FVIP
----------------------------------------------

Regarding the calculation of the radius, or the *depth* for the BMC and K-induction, it depends on some factors:
    * The ARM recommended properties for deadlock imposes a min radius of 16 plus extra cycles to let the solver explore more state space. If these properties are disabled, the second more complex property is the *channel relationships*. And of course, if the delay between the *ready* and *valid* signal is changed from 16, the bound should be fixed accordingly.
    * For the *channel relationships* and taking into account the features of this FVIP, the write transaction must complete before issuing another one, so the *depth should be sufficient to allocate enough time for this completion w.r.t the DUT*, plus some extra cycles to explore.
    * Therefore, the *default settings of SBY should be enough in most cases*, unless modifications to the already mentioned parameters are applied, to which the recommendations already described should be followed.

Our FVIP contains many cover properties to help decide if the depth is good enough (covers reached) or if it should be increased (unreachable covers).

K-induction with AXI SVA FVIP
-----------------------------
Everyone knows the equation of mathematical induction, but sadly not everyone seems to get what it really means for formal verification. To backup what I will write in this section, and hoping it helps to clear the doubts, look at the example that I did in 10 minutes (I'm not an artist, sorry for the bad drawing) which is located in the **Appendix A** if this document.

The real difficulties are to come with an inductive invariant. Remember that k-induction frees up the initial state, so a well defined, strong and complete set of assertions and correct initial values in registers, makes k-induction proofs happy. And the bound?, as discussed in **Appendix A**, can be as low as the employed inductive invariants permits. For the SVA AXI FVIP, the properties should not cause *undetermined* results in induction as long as the DUT is configured as expected (for example, that all the registers are correctly initialised). For advanced flows, the user can abstract this initial state and get the most of k-induction (as an example, in an interconenct verification, the user can abstract the initial state so the subordinates have many valid transactions pending, and check how the manager reacts from the first clock cycle).

Boolean Properties
------------------
Most properties are described using Boolean operators only, so all bit-level solvers are happy with them.

-------------------------------------------------------------
Appendix A. Simple and Oversimplified K-Induction Explanation
-------------------------------------------------------------

We want to play a game in this map. The goal is to get the treasure (depicted as dollar symbol) which is located in island D. But there are some rules that must be followed:
    * The game ends successfully when player reach **island D**.
    * The player must have passed through **island B** before reaching **island D**.
    * To travel from **island A** to **island B**, player needs to find the *purple mysterious box*. We know for a fact that the box is located in this **island A**.
    * Same rule applies for traveling from **island B** to **island C**, but the color of the box is *red* in this case.
    * Exactly the same rule applies for the path between **island C** to **island D**, but the color of the box is turquoise.
    * The player can take up to 3 months traveling between islands, because they are very far from each other.

+----------------------------------------------------------------------+
| .. image:: ../img/penup_20220416.jpg                                 |
|    :width: 6.5in                                                     |
|    :height: 2.93in                                                   |
|    :align: center                                                    |
+======================================================================+
| Figure 3.1. A map to induction.                                      |
+----------------------------------------------------------------------+

But there is another trick to help the player survive. Suppose the player can choose in which island to start, and in which condition they will be when starting in that island. The player in his ambition, decides to start immediately in **island B** and move through the blue bridge directly to the treasure. **They looses the game because they has no boxes to carry the treasure**.

The player gets a second chance, so they take a better look, and thinks that *if they visit island B correctly, is because they was in island A and got the purple box. And if they are in island C, <<assuming>> the first statement happened, and collect the turquoise box, then they can move to island D and get the treasure, and no rule is broken"*. So they decide to start in island C assuming they have visited previously island A and B, and have both purple and red boxes. In the first turn, the player gets the turquoise box, then moves to island D and wins the game.

How this relates with k-induction?
    * K-induction is like BMC, but freeing the initial state. That means, the solver can start at any state from the timeline of the design. In this example, the solver is analogous to the player, and the *free initial state* is the ability to start at any island.
    * Sometimes K-induction can return "weird" invalid results, because *the property has some holes*. Like in this example, the goal was reached when player was moving directly from island B to island D, but at the expense of not having fulfilled one requisite needed to win.
    *  The purpose of K-induction is to find inductive invariants, by strengthening the problem at hand:
        * The problem is to reach island D to get the treasure.
        * For **the basecase**, we assert that if island_A and purple_box follows island B and if island_B and red_box follows island C. If they are proven to be correct in this step, then we check the inductive step.
        * For the **inductive step**, we check that if island_C and turquoise box follows island_D and win. We *assume* the **basecase**, which lead us to only one path, which is the path we wanted to find. Then *we win*, because it does not matter from where the player starts, if the requisites are fulfilled, the player will end all the time reaching island D and wining. Also note that, since our property was strong enough, we rule out the initial path the player picked as starting point which led to losing the game (B to D using blue bridge).
        * This took no more than **2** steps to prove. Which means that a well defined inductive invariant does not need that many steps to be proven.





