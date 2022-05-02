===============================================
YosysHQ SVA AXI Verification IP
===============================================

========
Abstract
========
Due the popularity of AMBA protocols, AXI4 specifically, and trying to help our community to develop high quality designs with formal methods as easily as possible, we decided to create a Formal VIP for this protocol (yes, once again, but this time we tried to do it right). So in this AppNote we show you the results.

This AppNote will cover the following topics:
    * A quick glance at the newly refreshed (SVA-AXI4-FVIP)[https://github.com/YosysHQ-GmbH/SVA-AXI4-FVIP].
    * Parameters and configuration descriptions.
    * A usage example in an interconnect design.

============
Introduction
============
Everyone in the industry has heard about Verification IP (VIP), how these magical things can save
hours of development, testing and integration. Somehow, CAD vendors sweeten the ears of all of us [[not an idiom in english AFAIK, but still clear in meaning IMO]], who at
some point have needed to verify some complex component. They say, it's just as easy as connect it and tada!, all the
bugs are shown so engineers can fix them quickly.

Then, you get the VIP and collateral (**$omehow**) and you realize that:
    * The documentation is not very understandable.
    * There is barely any quick start example, or the one that is delivered is as trivial as it is useful.
    * It contains a lot of scripts, setup configurations and all sort of things that are not compatible
      with the configuration of the design that you are testing right now. So you need to do more work to
      integrate a component for the component that you want to integrate into your project.
    * Among lots of other things.

Where is the Return of Investment (ROI) and low Time To Market (TTM) that the company that sells
the VIP was talking about?.

Despite this bad first approach, you decide to go ahead and give it a chance, It can't be that bad, right?.
After pressing the <prove> button, the first counterexample (CEX) appears. By opening the wave to do root-cause analysis the first thing you see is this:

+----------------------------------------------------------------------+
| .. image:: ./media/encrypt.png                                       |
|    :width: 6.5in                                                     |
|    :height: 2.93in                                                   |
|    :align: center                                                    |
+======================================================================+
| Figure 1.1. A common CEX in a Formal VIP.                            |
+----------------------------------------------------------------------+

The assertion does not hold at clock cycle 5, and you cannot trace back *legal_wrap_pre* nor *legal_wrap_con*
signals because they are **encrypted**. Sometimes there is a message accompanying the property in the *fail statement* (where statements like *else*, *$info*, *$error*, etc, can be employed) that you can use to read the respective section of the specification where such behavior is described. For simple invariant (properties that holds despite input stimulus) such as handshakes, there is no complication. But for properties that depends on composed behaviors (data flow trackers like the CAM that most companies uses to track AXI IDs) it's another story. If you can't see what's going on inside, it's very easy to get lost: The AMBA spec says how things should behave, but not how everyone should build a VIP. The VIP tells you have a problem, but it doesn't help you to debug and fix it.

But, to be fair, not everything is the CAD companies fault. Developing a VIP is complicated. The engineers who write these VIPs needs to understand with great detail the specification, and exchange communications (emails) with the specification architects to clarify one thing or another. Speaking for example of the AMBA AXI and ACE Protocol Specification [missing link], almost everything is explained there, some things more detailed, some others miss  important information, or the second piece of info is indexed in an another part of the document or even worse, in another specification (try to read Atomic Accesses related sections knowing that any missing link is hidden in the Exclusive Monitor instructions (LDXR/STXR) programmer's guide, because this is ARM, and they share the same principles for most of their products).

============================
AXI4 Formal VIP Architecture
============================

--------
Features
--------
For more information regarding modes, properties descriptions and methodologies please check the *UG_verification_plan.pdf* file in the repository.

.. warning::
    The SVA AXI4 FVIP follows the Assume-Guarantee paradigm. This means that all results obtained from the formal verification tool are *guaranteed* *assuming* the driving (or neighbor) module drives protocol-compliant signals as well. So it is recommended to verify both the driver and the receiver for completeness.

The features and limitations of the new AXI4 SVA FVIP are the following ones:
    * Support for different usage models:
        * To verify a manager.
        * To verify a subordinate.
        * To act as a protocol-compliant constraints provider.
        * To act as a monitor in a bus where a manager and a subordinate exists.
    * Support for AXI4-Lite and AXI4-full.
    * Single transaction ID.
        * In-order transactions.
        * No interleaving.
    * No transaction caching, buffering or modifications.
        * Same bus width on all interfaces.
    * Written in simple SystemVerilog constructs for portability.
    * ARM recommended checks:
        * Low power channel interface.
        * X-propagation interface. [[mention that this is not supported by tabby yet?]]
        * Deadlock checks.
    * Comprehensible documentation.
        * User guide.
        * Methodology guide.
        * Examples.
        * Instantiation templates.

.. note::
    Full AXI4 implementation is possible. In fact, at the moment of writing this AppNote, we have the capacity to test more than one transaction at a time, out-of-order transactions, full exclusive transaction monitors, data interleave, etc. But for the purpose of simplicity, and because these features cover most of the cases, we decided to release the IP in this state. The released FVIP has the required logic to add these features easily, which is another advantage of the open source components.

------------
Architecture
------------
We designed the AXI4 SVA FVIP keeping in mind the fundamental architectural descriptions in the AMBA AXI4 IHI0022E spec (A1.3 AXI Architecture):
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
        * That means we did not just define things based on our interpretation of the descriptions in the spec, but followed them strictly. [[ I guess this is trying to say that it's not based on a superficial loose interpretation? But in the end it's still based on our interpretation, which the note below also says. It's a bit confusing in the current state. ]]
        * We developed an infrastructure to verify our implementation based on information that is publicly available from the ARM website.

.. note::
    We are an small company, we have no partnership with ARM. If there is any misinterpretation please let us know, but at the moment we have no seen any divergence between results of public ARM verification IP and ours.

The *Figure 2.1* shows the architecture of the AXI4 SVA FVIP. For more information refer to the *UG_verification_plan, Section 6 Architecture*.

+----------------------------------------------------------------------+
| .. image:: ./media/org.png                                           |
|    :width: 6.5in                                                     |
|    :height: 2.93in                                                   |
|    :align: center                                                    |
+======================================================================+
| Figure 2.1. Architecture and file organisation.                      |
+----------------------------------------------------------------------+

As an example of what is described above, this is the `valid_before_handshake` property defined inside *amba_axi4_single_interface_requirements.v* package, which is derived from section A3 of the AMBA AXI4 spec. All properties described in that section are defined in the same package.

.. code-block:: systemverilog

   /* ,         ,                                                     *
    * |\\\\ ////| "Once VALID is asserted it must remain asserted     *
    * | \\\V/// |  until the handshake occurs, at a rising clock edge *
    * |  |~~~|  |  at which VALID and READY are both asserted".	      *
    * |  |===|  |  Ref: A3.2.1 Handshake process, pA3-39. 	      *
    * |  |A  |  |						      *
    * |  | X |  |						      *
    *  \ |  I| /						      *
    *   \|===|/							      *
    *    '---'							      */
   property valid_before_handshake(valid, ready);
      valid && !ready |-> ##1 valid;
   endproperty // valid_before_handshake

Then, in each channel that needs to honor this property, it is assembled as shown below:

.. code-block:: systemverilog

    if(cfg.VERIFY_AGENT_TYPE inside {SOURCE, MONITOR}) begin
         ap_W_AWVALID_until_AWREADY: assert property(disable iff(!ARESETn) valid_before_handshake(WVALID, WREADY))
           else $error("Violation: Once WVALID is asserted it must remain asserted until the handshake",
                       "occurs (A3.2.1 Handshake process, pA3-39).");
      end
      else if(cfg.VERIFY_AGENT_TYPE inside {DESTINATION, CONSTRAINT}) begin
         cp_W_AWVALID_until_AWREADY: assume property(disable iff(!ARESETn) valid_before_handshake(WVALID, WREADY))
           else $error("Violation: Once WVALID is asserted it must remain asserted until the handshake",
                       "occurs (A3.2.1 Handshake process, pA3-39).");
      end

The user can drag and drop the signals to the waveform, only the ones stated in the property, and look at the message and/or the package where this property is defined to start debugging. Sometimes, the message in the assertion is so clear that there might be not need to lookup at the spec, but never trust code, it is recommended to confirm the relevant reference.

===================================================
Formalisation and Optimisation of the AXI4 SVA FVIP
===================================================

------------------------------
When to use BMC or K-induction
------------------------------
All of the properties defined in the IHI0022E spec are invariants, that is, they must hold *invariably* of the design input values and/or initial states. A good rule of thumb is to use *BMC* for the AXI control signals, such as handshakes, strobes, etc, and start with BMC but move incrementally to K-induction for data transport checks, such as properties for *channel relationships* or whenever tracking of "in-flight" data is needed. Although BMC with sufficient depth [[ not sure if the terminology is used slightly different in practice, but AFAIK radius is the depth needed to check all reachable states, so I think this should be depth ]] can be enough to gain confidence.

Bounded Model Checking (BMC) with AXI SVA FVIP
----------------------------------------------
Regarding the calculation of the radius, or the required *depth* for the BMC and K-induction, it depends on some factors:
    * The ARM recommended properties for deadlock imposes a min radius of 16 plus extra cycles to let the solver explore more state space. If these properties are disabled, the second more complex property is the *channel relationships*. And of course, if the delay between the *ready* and *valid* signal is changed from 16, the bound should be fixed accordingly.
    * For the *channel relationships* and taking into account the features of this FVIP, the write transaction must complete before issuing another one, so the *depth should be sufficient to allocate enough time for this completion w.r.t the DUT*, plus some extra cycles to explore.
    * Therefore, the *default settings of SBY should be enough in most cases*, unless modifications to the already mentioned parameters are applied, in which case the recommendations already described should be followed.

Our FVIP contains many cover properties to help decide if the depth is good enough (covers reached) or if it should be increased (unreachable covers).

K-induction with AXI SVA FVIP
-----------------------------
Everyone knows the equation of mathematical induction, [[ one would hope ^^, I generally avoid such statements though, if someone knows this, it doesn't add anything, and if someone doesn't know this, they might feel uneceserrily discouraged from reading on and/or using the FVIP ]] but it's proven difficult to really understand what it means for formal verification. To help explain further, look at the example drawing located in the **Appendix A** if this document.

The real difficulties come with an inductive invariant. Remember that k-induction frees up the initial state [[ is "free up" established terminology? I would probably go with "does not constrain" or "does not restrict" but I'm not sure which is better for the target audience. ]], so a well defined, strong and complete set of assertions and correct initial values in registers, makes k-induction proofs happy. And the depth?, as discussed in **Appendix A**, it can be as low as the employed inductive invariants permits. For the SVA AXI FVIP, the properties should not cause *undetermined* results in induction as long as the DUT is configured as expected (for example, that all the registers are correctly initialised). For advanced flows, the user can abstract this initial state and get the most out of k-induction (for example, in an interconnect verification, the user can abstract the initial state so the subordinates have many valid transactions pending, and check how the manager reacts from the first clock cycle).

As with BMC, the default configuration of SBY may be enough for most of the cases, and modifications would be needed only if different parameters or the design changes in complexity.

------------------
Boolean Properties
------------------
Most properties in the AXI SVA FVIP are described using Boolean operators, so all bit-level solvers are happy with them. We wanted to explore some things using the SMT solvers technology in [TabbyCAD](https://www.yosyshq.com/products-and-services), but after some struggles with other users and tools, we decided to keep this as simple as possible. [[ this reads a bit strange to me, I think it's easy to misread this as "there are issues with more advanced SMT solver features in TabbyCAD" even though it's not what it is saying. ]]

------------------------
Data Tracking Invariants
------------------------
Control properties are easy to describe in the AXI4 protocol, what is more tricky is to formalise the properties where data tracking is required, for example, atomic transactions and dependencies between channels. We will use the latter as an example for this section.

The AMBA AXI4 IHI0022E depicts the channel dependencies with the following data flow diagram:

+----------------------------------------------------------------------+
| .. image:: ./media/interdep.png                                      |
|    :width: 6.5in                                                     |
|    :height: 2.93in                                                   |
|    :align: center                                                    |
+======================================================================+
| Figure 3.1. Channel dependencies in AXI4.                            |
+----------------------------------------------------------------------+

What this means in short is, for a subordinate to show a *valid response*, the following events must have happened:
    * A valid address write, signalled by the completion of the AW channel (AWVALID & AWREADY handshake).
        * Here, we store the AWID, the tag of such transaction.
    * Of course, the data of such an address request must have completed as well (completion signalled by the handshake of WVALID & WREADY).
        * A very important item of information here is that *WLAST* should occur first before asserting *WVALID*, so when we have a handshake in the W channel, we store the WLAST value as well.
    * Finally, whe monitor for the assertion of *BVALID*, to check the following properties (they are split for convergence/performance reasons).
        * The value at *BID* must match one of the stored values of AWID (in the case of OOO transactions) or the value stored in the head of the data structure (in case of in-order transactions). Otherwise the response is invalid.
        * The value of WLAST stored during the W transaction must be HIGH, otherwise the response is invalid.

This is how we cover the dependencies between AW, W and B channels, as the rest of scenarios where different order of handshakes can occur needs to fulfill this rule anyway (these scenarios can be observed with a cover property, but is a mere preference of the visualization information this bring to the user, so we decided to not add them).

To track data, much AXI simulation IP uses CAM-based tables, which is an obvious solution, but since it searches the entire table for the stored ID, this becomes a burden for formal verification (the more IDs, the more states the CAM adds to the model). Our solution is to use a non-deterministic transaction-counter structure which has the following features:
    * Implicit forward-progress counters: one can see how many transactions are pushed into the pipeline, how many are read, or if there is no transactions at all.
    * Deadlock checking: each transaction is marked with a timestamp (in clock cycles) to put a constraint on the life of such transactions. If the transfer is not processed and reaches timeout, the scoreboard signals an error for further investigation (either deadlock or performance issue).
    * And of course, data integrity checks for the stored IDs.

The disadvantage of this approach is that the user should know beforehand, the maximum number of transactions the IP can handle. We recommend to start tracking a low number of transactions and to incrementally increase the number.

The figure 3.2 shows how the scoreboard works. As soon as the AW handshake occurs, the value seen at AWID is stored. In this example, we store two AWIDs with values :systemverilog:`'h00`' and :systemverilog:`'hFF`. Once a pipeline packet has stored a transfer, we mark it as active. When BVALID is asserted, the value presented at BID must match the value stored at the head of the pipeline data structure. If this is the case, the behavior is proven, otherwise a CEX is shown. Once a packet has been read, we mark it as invalid.

+----------------------------------------------------------------------+
| .. image:: ./media/scoreboard.png                                    |
|    :width: 6.5in                                                     |
|    :height: 2.93in                                                   |
|    :align: center                                                    |
+======================================================================+
| Figure 3.2. The scoreboard functionality.                            |
+----------------------------------------------------------------------+

.. note::
    * The counters at *timeout* can be used to get an idea of the performance of the DUT. The timeout checks can be disabled.
    * There is an overflow check that is asserted when more write requests than pipeline packets exists. This can be disabled as well.
    * by looking at how many packets become active/inactive, we can see that we actually make progress during transaction verification, and that no check is vacuous.

=======================
Using the SVA AXI4 FVIP
=======================
The SVA AXI4 FVIP comes with some basic examples that we describe in this section.

--------------
Synthesis Test
--------------
The most basic and fundamental way to test a formal verification IP is by the tautology method, that is, connecting the assertions to their versions as assumptions. If everything is configured correctly, all checks should pass within seconds. If there is some misconfiguration, or something that exists as a check but not as a constraint, or vice versa, the tool will show a CEX.

This test is much more useful when comparing different implementations, for example, comparing FVIP from vendor *A* to the FVIP from vendor *B*.

Whenever the user adds new properties or modifications, it is recommended to run this test before running the test directly on the DUT.

------------------
AMBA Validity Test
------------------
This test uses the AMBA certified SVA IP (intended for simulation) as reference to check the validity and satisfiability of the YosysHQ AXI4 SVA FVIP. This test is just a bounded model check [[I presume]] between formal IP assumptions and formal IP assertions, using the AMBA SVA IP as a monitor agent. The results are interpreted as follows:

    * Any assertion that passes in the AXI4 SVA FVIP but not in the AMBA IP, may be a failure.
    * Any assertion that fails in the AMBA IP, is either a failure or a missing behavior.

Users can check the `Results.xlsx` sheet that contains the latest results from this test.

-----------------------------
SpinalHDL AXI4-Lite Component
-----------------------------
For this example, we use [SpinalHDL](https://github.com/SpinalHDL/SpinalHDL) to write a very simple AXI4-Lite component. We are not interested in the datapath but in the control,  therefore the actual function that the scala source describes is not relevant. Here is an excerpt of such component.

.. code-block:: scala

    class AxiLite4FormalComponent extends Component {
        val io = new Bundle {
        val bus = slave (AxiLite4 (AxiLite4Config (addressWidth = 32, dataWidth = 32)))
        val o_result = out UInt (32 bits)
    }

      val ctrl = new AxiLite4SlaveFactory (io.bus)
      var AxiFunction = new LogicFunction ()
      ctrl.driveAndRead (AxiFunction.io.port_a, address = 0)
      ctrl.driveAndRead (AxiFunction.io.port_b, address = 4)
      ctrl.read (AxiFunction.io.port_r, address = 8)

      io.o_result := AxiFunction.io.port_r
    }

There are some protocol violations in this design. For example, the property *ap_AR_STABLE_ARPROT* is violated, as **ARPROT** can change its value when it has not been acknowledged (red shows the violation).

+----------------------------------------------------------------------+
| .. image:: ./media/spinal_arprot.png                                 |
|    :width: 6.5in                                                     |
|    :height: 2.93in                                                   |
|    :align: center                                                    |
+======================================================================+
| Figure 4.1. A simple CEX found in the example.                       |
+----------------------------------------------------------------------+

The SBY gui can be launched by executing the command *sby-gui* where the ***.sby** file reside, in this case in *AXI4/examples/spinal_axi4_lite/*.

-------------
AXI4 Crossbar
-------------
We also provide an example of how to use the FVIP to test different configurations for crossbars/interconnects. In more complex designs where different topologies are involved, or even where different types of bridges and adapters are required, but testing the entire system become very complex, the FVIP can be used to replace the upstream/downstream components to focus on one task at a time. Figure 4.2 shows a diagram of how the FVIP is connected to the crossbar.

+----------------------------------------------------------------------+
| .. image:: ./media/arch_xbar.png                                     |
|    :width: 6.5in                                                     |
|    :height: 2.93in                                                   |
|    :align: center                                                    |
+======================================================================+
| Figure 4.2. Crossbar verification architecture.                      |
+----------------------------------------------------------------------+

There is a document that covers the setup and some results of this example in *AXI4/examples/axi_crossbar/doc/crossbar_example.pdf*. One of the properties that failed is the *Read burst crossing 4K address boundary*. The AXI4 Formal IP found a violation in the crossbar around time step 19, **ARBURST = INCR**, **ARLEN = 1Ch**, **ARSIZE = 1h** and **ARADDR = 1EFE3h** giving a final address of **1F01Bh**, crossing the 4K boundary.

+----------------------------------------------------------------------+
| .. image:: ./media/ar_bound_4k.jpg                                   |
|    :width: 6.5in                                                     |
|    :height: 2.93in                                                   |
|    :align: center                                                    |
+======================================================================+
| Figure 4.3. Simple CEX found in the crossbar.                        |
+----------------------------------------------------------------------+

.. note::
    The failing property was obtained in the inductive test and may not be valid, but it has a purpose. One usually can find interesting scenarios by weakening the inductive property (not adding all required constraints but with some guidance), because SBY cannot generate certificates of witness yet, so this can help to investigate the design further. This is not a recommendation, and many times it does not serve a purpose without having previous knowledge of certain weak structures of the design. [[ look at this in more detail, I'm not sure it is saying what I think it is, might be clear with more context, but maybe it can also be improved ]]

===============================
Concerns with the AXI4 Protocol
===============================

---------------------------
Problems with the handshake
---------------------------

There are some widely know behaviors not covered in the AMBA AXI4 IHI0022E spec, the most popular one is *the  strong dependency in the VALID signals of the handshake*. There are some studies out there [[this would be a wonderful place to cite these :)]] that showcase how data can be exposed without any of the assertions being fired, by injecting Trojans that allows data extraction when VALID is deasserted (because no assertions to check what should happen in this scenario exist).

.. code-block:: systemverilog

    property stable_before_handshake(valid, ready, control);
      valid && !ready |-> ##1 $stable(control);
   endproperty // stable_before_handshake

If the valid signal is deasserted, the property passes vacuously. Indexed relationships to this behavior can be undetected as well (checks for strobes, responses, IDs, etc). It is recommended to make sure nothing goes bad when VALID signals are deasserted, specially with security oriented IPs and/or atomic operations.

----------------------------
The Transaction Dependencies
----------------------------
Previous versions of AMBA AXI protocol allowed to signal *BRESP* after *WLAST*, without requiring the *AW* channel to complete. This sometimes created confusions as to which transaction the subordinate should respond. For AXI4, the requirement of *AW* and *WLAST* before *BRESP* is explicitly stated in the specification, therefore we enforce this check in the FVIP instead of making it optional, but can be disabled if this causes some unwanted complexity.

----------------------------------
Implementation Dependent Behaviors
----------------------------------
Some behaviors are not defined in the AMBA AXI4 IHI0022E spec, specially for interconnects. This may cause problems for some implementation dependent in cases such as interleaved transactions and atomic accesses. For example, in previous revisions of the spec there was a misunderstanding for designs that allows AW and AR transactions with equal values of the ID at the same ACLK, this could cause violations to the atomic accesses definition. In the latest revisions of the spec this property is enforced, so we also added it to the FVIP as non-optional. It can be disabled if, for example, no atomic accesses are supported in the design.

.. code-block:: systemverilog


   /* ,         ,                                                     *
    * |\\\\ ////|  "A master must not start the write part of an      *
    * | \\\V/// |   exclusive access sequence until the read part is  *
    * |  |~~~|  |   complete".                                        *
    * |  |===|  |   Ref: A7.2.2 Exclusive access from the perspective *
    * |  |A  |  |        of the master, pA7-96.                       *
    * |  | X |  |                                                     *
    *  \ |  I| /                                                      *
    *   \|===|/                                                       *
    *    '---'                                                        */
   /* This is an important property for crossbars as far as I'm
    * concerned. What I don't understand is why AMBA DUI0534 formalises
    * this property overly complex. I am missing something?.
    * Allow me to explain: if a system should not start AW in
    * EXCLUSIVE with R in EXCLUSIVE, it is not sufficient to check
    * that these two conditions never happens at the same time? ...
    * We'll figure it out in the next test of `amba_validity_test`. */
   sequence exclusive_write_read(avalid, aid, alock, arvalid, arid, arlock);
      aid == arid          ##0
      avalid               ##0
      arvalid              ##0
      alock  ==  EXCLUSIVE ##0
      arlock == EXCLUSIVE;
   endsequence // exclusive_write_read

   property exclusive_wr_rd_simultaneously;
      (not (exclusive_write_read(AWVALID, AWID, AWLOCK, ARVALID, ARID, ARLOCK)));
   endproperty

Atomic access are not very well described in the specification, so there might be some other scenarios that can cause bad interpretation. For example, the spec is clear on how to drive AxPROT and AxCACHE during the handshake, but not how they should behave during for multiple locked transactions. The seasoned engineer can reach a conclusion quite fast, but then again this is based on interpretation.

.. note::
    It is strongly recommended to run some type of coverage. For example, assertions that are not activated with mutations in MCY can lead to discovering holes in the design, even when using the SVA AXI4 FVIP.


===========
Conclusions
===========
AXI4 is a widely used bus protocol for the industry and hobbyists, so it makes sense to have some solutions to easily verify its correctness. We do not promise our FVIP will do miracles, what we promise is that the user will have the means to audit the code, remove unwanted proofs, add some checks on top of the ones available right now, and/or explore advanced flows with the FVIP such as security and reliability of designs.

The results we have seen right now are good enough to be released [[I think it would be better to end with something more positive than "good enough", maybe something like "the results we have seen so far are already promising, so we decided to release this AppNote." the paragraph above already says no miracles included ]], that is why we are writing this AppNote. In case of doubts with the implementation, questions regarding the AXI4 protocol, or support request, you can open an issue in the GitHub repository.
