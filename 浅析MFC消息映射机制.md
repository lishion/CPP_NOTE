# 浅析MFC消息映射机制

## MFC消息映射概述

在MFC中消息的处理方式与直接使用API方式有很大的区别,MFC利用**消息映射机制** 将消息和响应的方法映射起来以达到简化消息处理的目的。本文将从消息的结构、消息映射的方式以及实现的细节来对MFC的消息映射机制进行不太深刻的理解。

### 消息的结构

MFC中的消息用一个结构体来表示，这个结构体放在**afxwin.h** 中。结构体的申明如下:

`struct AFX_MSGMAP_ENTRY
{

```c++
UINT nMessage;   // windows message
UINT nCode;      // control code or WM_NOTIFY code
UINT nID;        // control ID (or 0 for windows messages)
UINT nLastID;    // used for entries specifying a range of control id's
UINT_PTR nSig;       // signature type (action) or pointer to message #
AFX_PMSG pfn;    // routine to call (or special value)
```
};`

从代码可以看出该结构体的的参数有下面几个:

1. nMessage 表示消息的类型 MFC中有系统消息、窗体通知和一般消息等。

2. nCode 控制消息的消息码

3. nID 如果是控件消息 则表示控件的ID

4. nLastID 如果有多个控件消息，则nID表示起始ID，nLastID表示终止ID

5. nSig 表示消息处理函数的标识，后面会提到作用

6. pfn是指向消息处理函数的函数指针，AFX_PMSG表示一种无参无返回值的类型。定义在**afxwin.h** 中，定义如下：

   `typedef void (AFX_MSG_CALL CCmdTarget::*AFX_PMSG)(void)`

同时，MFC还定义了另一结构体来关联不同的类之间消息：

`struct AFX_MSGMAP
{

	const AFX_MSGMAP* (PASCAL* pfnGetBaseMap)();
	const AFX_MSGMAP_ENTRY* lpEntries;
};`

其中pfnGetBaseMap指向另一个类的AFX_MSGMAP，lpEntries是一个指向AFX_MSGMAP_ENTRY类型构成的数组的指针。

## 消息映射的方式 

谈到消息映射的方式，就不得不在看一下**AFX_MSGMAP_ENTRY** 这个结构体，这个结构体中存放了关于一条消息的类型，ID，已经还有响应该消息的方法的指针(即回调函数的指针)，通过这种方式就能将不同的消息和不同的响应函数映射起来。但是上面提到过回调函数的指针是一个**无参无返回值** 类型的指针，所以在对不同的回调函数进行映射的过程中都会像转换为这种类型的函数指针，但是为什么MFC要这么设计呢？是因为在开发的过程中，不同的消息对应着不同的回调函数。有无参的，有一个参数的，有两个参数的。所以无法设计一个万能的函数指针来保存这些回调函数的信息，就干脆设计一个无参无返回值的函数指针达反而更方便。到这里又会产生疑问，一个回调函数被转为无参无返回值的函数了，那真正调用它的时候又该怎么办呢？是要装回去吗？那如果要转回去又应该根据什么来转呢？这里MFC使用了UNION共用体来调用真正的回调函数，有一定技巧性。

## 具体的实现

大家谈到MFC的消息映射机制，一般都会想到**BEGIN_MESSAGE_MAP** 、**END_MESSAGE_MAP** 等几个宏，一般的菜鸟看到这几个宏都是敬而远之，往往不会去深究它的原理。我也是其中的一个，毕竟在我看过的所有的C++的书籍中都有*能不用宏就不用宏、C++根本就不需要用宏、宏是C语言设计的缺陷* 等几种警告性的提示。导致我看到这几个宏就觉得这一点是设计者才能搞懂的东西，我们一般的小白还是不要了解的好。但是，反正那里总觉得还是应该大概的看一下其作用，于是才有了这篇文章。这几个宏在**afxwin.h** 中，由于其中有宏定义会出现一级标题我就不贴了，放几句比较有代表性的：

`static const AFX_MSGMAP_ENTRY _messageEntries[]={0, 0, 0, 0, AfxSig_end, (AFX_PMSG)0 }`

 实现消息映射的主要有ON_BN_COMMOND这个宏来实现，其中COMMOND表示具体的命令，比如鼠标单击的控件消息映射如下:

`ON_BN_CLICKED(IDC_SELECT_PICTURE_BUTTON, &Cimage_enchanceDlg::OnBnClickedSelectPictureButton)`

其中IDC_SELECT_PICTURE_BUTTON是控件的ID，Cimage_enchanceDlg::OnBnClickedSelectPictureButton是对应的回调函数

将这个宏展开后是这样的:

`{ WM_COMMAND, (WORD)wNotifyCode, (WORD)id, (WORD)id, AfxSigCmd_v, (static_cast< AFX_PMSG > (memberFxn)) }`

其中id就是控件的ID,这里看到有两个一样的ID表示发出消息的控件只有一个，所以两个ID是一样的。memberFxn表示的就是回调函数的指针，确实被转换为AFX_PMSG 类型的。而这整个就构成了一个**AFX_MSGMAP_ENTRY** 结构体，将多个这样的消息放在一起就构成了一个消息数组，最后响应的时候会在这个数组中进行查找，找到响应的消息后就会调用相应的回调函数。这样就是MFC消息映射机制的一个简要的描述。









