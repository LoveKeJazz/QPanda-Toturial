量子程序匹配拓扑结构
=====================

量子计算设备存在量子比特之间的有限连接，使得只能在有限的量子位对上应用两个量子位门。
量子程序应用到目标设备时，必须转换原始的量子程序以适应硬件限制，让双量子比特门中的两个量子比特能够满足物理拓扑结构，从而让双量子位门正常作用；
当前解决方案中多数需要在无法相互作用的两个量子比特间插入额外的SWAP操作，以便将逻辑量子位“移动”到它们可以相互作用的位置。
我们称这种解决方法为量子程序匹配拓扑结构。
当前Qpanda2版本中存在两种思路的匹配拓扑方法：

- 接口  ``QProg topology_match(QProg prog, QVec &qv, QuantumMachine * qvm, SwapQubitsMethod method, ArchType arch_type)``
   通过采用线路分层以及A*搜索算法，在匹配过程中，让插入的SWAP操作个数近似达到最少，使得算法的整体近似消耗达到最少。

- 接口  ``QProg qcodar_match(QProg prog, QVec &qv, QuantumMachine * qvm, QCodarGridDevice arch_type, size_t m, size_t n, size_t run_times)``
   一种上下文敏感和持续时间感知的重映射算法，该算法能够感知门持续时间差和程序上下文，使得它能够从程序中提取更多的并行性，
   在不同体系结构的模拟中平均将量子程序的速度提高1.23，并在原始量子噪声模拟器上运行时保持电路的保真度。

接口详细说明如下：

.. code-block:: c
        /**
        * @brief  QProg/QCircuit matches the topology of the physical qubits
        * @ingroup Utilities
        * @param[in]  QProg  quantum program
        * @param[in]  QVec  qubit  vector
        * @param[in]  QuantumMachine *  quantum machine
        * @param[in]  SwapQubitsMethod   swap qubits by CNOT/CZ/SWAP/iSWAP gate
        * @param[in]  ArchType    architectures type
        * @return    QProg   mapped  quantum program
        */
        QProg  topology_match(QProg prog, QVec &qv, QuantumMachine *machine,
            SwapQubitsMethod method = CNOT_GATE_METHOD, ArchType arch_type = IBM_QX5_ARCH);

.. code-block:: c
        /**
        * @brief   A Contextual Duration-Aware Qubit Mapping for V arious NISQ Devices
        * @ingroup Utilities
        * @param[in]  QProg  quantum program
        * @param[in,out]  QVec  qubit  vector
        * @param[in]  QuantumMachine*  quantum machine
        * @param[in]  QCodarGridDevice   grid device type
        * @param[in]  size_t   m : the length of the topology,  if SIMPLE_TYPE  you need to set the size
        * @param[in]  size_t   n  : the  width of the topology,   if SIMPLE_TYPE  you need to set the size
        * @param[in]  size_t   run_times  : the number of times  run the remapping, better parameters get better results
        * @return    QProg   mapped  quantum program
        * @note	 QCodarGridDevice : SIMPLE_TYPE  It's a simple undirected  topology graph, 
        *                                        build a topology based on the values of m and n
        */
        QProg qcodar_match(QProg prog, QVec &qv, QuantumMachine * machine, 
            QCodarGridDevice arch_type = SIMPLE_TYPE, size_t m = 2, size_t n = 4, size_t run_times = 5);


实例
---------------

.. code-block:: c

    #include "Core/Core.h"
    using namespace std;
    using namespace QPanda;
    int  main()
    {
        auto qvm = new CPUQVM();
        qvm->init();
        auto q = qvm->allocateQubits(8);
        auto c = qvm->allocateCBits(8);
        auto srcprog = QProg();
        srcprog << CNOT(q[0], q[3])
            << CNOT(q[0], q[2])
            << CNOT(q[1], q[3])
            << CZ(q[1], q[2])
            << CZ(q[0], q[2])
            << T(q[1])
            << S(q[2])
            << H(q[3]);

        qvm->directlyRun(srcprog);
        auto r1 = qvm->PMeasure_no_index(q);

        QProg  outprog = topology_match(srcprog, q, qvm, CNOT_GATE_METHOD, ORIGIN_VIRTUAL_ARCH);

        qvm->directlyRun(outprog);
        auto r2 = qvm->PMeasure_no_index(q);

        int size = std::min(r1.size(), r2.size());
        bool result_equal = true;

        for (int i = 0; i < size; i++)
        {
            if ((fabs(r1[i] - r2[i]) > 1e-6))
                result_equal = false;
        }

        if (result_equal == true)
        {
            std::cout << "The probability measurements are the same, prob list:  " << std::endl;
            for (int i = 0; i < size; i++)
            {
                std::cout << r1[i]  << std::endl;
            }
        }

        qvm->finalize();
        delete qvm;
        return 0 ;
    }


运行结果如下：
::
    The probability measurements are the same, prob list:
    0.5
    0
    0
    0
    0
    0
    0
    0
    0.5
    0
    0
    0
    0
    0
    0
    0


.. code-block:: c
    #include "Core/Core.h"
    using namespace std;
    using namespace QPanda;
    int main()
    {
        auto  qvm = new CPUQVM();
        qvm->init();
        auto q = qvm->allocateQubits(4);
        auto cv = qvm->allocateCBits(4);
        QProg prog;
        prog << CNOT(q[1], q[3])
            << RX(q[0], PI / 2)
            << CNOT(q[0], q[2])
            << CNOT(q[1], q[3])
            << RY(q[1], -PI / 4)
            << CNOT(q[2], q[0])
            << CZ(q[1], q[2])
            << CNOT(q[1], q[3])
            << RZ(q[2], PI / 6)
            << CNOT(q[2], q[0])
            << RZ(q[0], -PI / 4)
            << CNOT(q[0], q[2])
            << H(q[0])
            << T(q[1])
            <<RX(q[1], -PI/4)
            << Y(q[2])
            << Z(q[1])
            ;

        qvm->directlyRun(prog);
        auto r1 = qvm->PMeasure_no_index(q);
        QProg out_prog = qcodar_match(prog, q, qvm, SIMPLE_TYPE, 2, 3, 5);

        qvm->directlyRun(out_prog);
        auto r2 = qvm->PMeasure_no_index(q);

        int size = std::min(r1.size(), r2.size());
        bool result_equal = true;

        for (int i = 0; i < size; i++)
        {
            if ((fabs(r1[i] - r2[i]) > 1e-6))
                result_equal = false;
        }
        if (result_equal == true)
        {
            std::cout << "The probability measurements are the same, prob list:  " << std::endl;
            for (int i = 0; i < size; i++)
            {
                std::cout << r1[i] << std::endl;
            }
        }
        qvm->finalize();
        delete qvm;
        return 0;
    }


运行结果如下：
::
    The probability measurements are the same, prob list:
    0
    0
    0
    0
    0.269995
    0.458558
    0.0463238
    0.0786762
    0
    0
    0
    0
    0.0134987
    0.00794791
    0.0786762
    0.0463238