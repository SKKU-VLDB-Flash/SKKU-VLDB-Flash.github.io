---
layout: default
title: Overview & Spec
parent: OpenChannel SSD
nav_order: 1
---

# OpenChannel SSD NVMe 


- [Theory of Operation](https://skku-vldb-flash.github.io/docs/OCSSD/OCSSD_spec.html#theory-of-operation)
	- [Overview](https://skku-vldb-flash.github.io/docs/OCSSD/OCSSD_spec.html#overview)
	- [Architecture](https://skku-vldb-flash.github.io/docs/OCSSD/OCSSD_spec.html#storage-software-stack) 
	- [SSD Representation](https://skku-vldb-flash.github.io/docs/OCSSD/OCSSD_spec.html#open-channel-ssd-representation)
- [PPA Command Set](https://skku-vldb-flash.github.io/docs/OCSSD/OCSSD_spec.html#physical-page-address-command-set)

---


## Theory of Operation

### Overview
Open-Channel SSD (OCSSD) 는 기존의 SSD 내부의 펌웨어 상에서 완전히 관리되어 왔던 기능들을 호스트와 공유하는 디바이스를 말한다. OCSSD 는 SSD 내부의 병렬구조를 호스트에 노출함으로써 I/O isolation, predictable latency, 그리고 호스트 레벨에서의 메모리 관리 등이 가능케 한다.

#### I/O Isolation 
OCSSD 는 SSD의 물리적인 영역을 여러 개의 PU (Parallel Unit 으로 나눈다 - LUN in v.1.2) 로 나눈다. I/O isolation 은 디바이스의 PU 에 매핑되어있는 논리적 단위로 SSD 용량을 분리하였기 때문에 가능하다. 특히 이를 활용하여 멀티태넌트(multi-tenant) 응용 환경에서 호스트는 다른 태넌트에 의하여 영향받지 않도록 조정할 수 있다. 

#### Predictable Latency
OCSSD 에서는 호스트가 직접 I/O 커맨드를 언제, 어디에 그리고 어떻게 전송할지 결정할 수 있기 때문에 latency 를 예측할 수 있다. PU 간의 IO isolation 이 서로 다른 PU 의 간섭을 막고, PU 가 active 한지 아닌지를 할 수 있기 때문에 latency 의 예측이 가능하다.

#### Host-Managed Non-Volatile Memory Management
호스트가 데이터의 위치를 결정할 수 있으며, IO 의 스케줄링이나 파일시스템/응용과의 밀결합이 가능하다.

### Storage Software Stack

#### Device Driver
OCSSD PPA 명령 셋을 지원하는 디바이스 드라이버로 PPA 명령 셋에 대해서는 아래에서 설명하겠다.

#### Media Manger 
미디어 매니저는 유저와 물리장치 사이의 중개자 역할을 한다. LightNVM 의 미디어 매니저는 최소한의 블 관리를 담당하며 타켓이나 유저로 하여금 실제 FTL 과 같이 동작할 수 있도록 한다. 따라서 미디어 매니저는 데이터의 위치 선정, 가비지 콜렉션, 블록 관리 등을 담당한다. OCSSD 2.0 버전으로 가면서 SSD 중심의 일부 기능들이 제거되어 [lightnvm.h](https://github.com/OpenChannelSSD/linux/blob/master/include/linux/lightnvm.h) 의 Geometry 구조체에서 미디어 타입과 밀접한 연관이 있는 부분들은 backward compatibility 를 위해서만 남아있다.

#### Target
General 미디어 매니저를 사용하는 경우 target 은 FTL 기능을 제공한다. 뿐만 아니라 유저 스페이스와 소통할 수 있는 인터페이스도 제공한다. (e.g., [pblk](https://github.com/OpenChannelSSD/linux/tree/master/drivers/lightnvm))

뿐만 아니라 LightNVM 은 유저스페이스 라이브러리인 [liblightnvm](https://github.com/OpenChannelSSD/liblightnvm) 을 통해서 응용 레벨에서 FTL 을 관리하는 응용이 get/put 블록 인터페이스를 제공하기도 한다.


### Open-Channel SSD Representation 

<div class="code-example" markdown="1">
아래 코드는 [lightnvm](https://github.com/OpenChannelSSD/linux/blob/master/include/linux/lightnvm.h) 의 물리적 구조를 보여준다. 1.2 버전에서와 2.0 버전의 주소체계에 차이가 있음을 확인할 수 있는데 사실상 아래의 예시에서 볼 수 있듯이 대응된다.
- channel-grp: bus 로 연결된 PU 의 그룹
- LUN-PU: 병렬 유닛
- BLK-CHK: erase 유닛 (2.0 버전에서는 reset 한다고 표현함.)
</div>
{% highlight markdown %}
struct ppa_addr{
	union {
		.
		.
		.
		/* 1.2 device format */
		struct {
			u64 ch		: NVM_GEN_CH_BITS;
			u64 lun		: NVM_GEN_LUN_BITS;
			u64 blk		: NVM_GEN_BLK_BITS;
			u64 pg		: NVM_12_PG_BITS;
			u64 pl		: NVM_12_PL_BITS;
			u64 sec		: NVM_12_SEC_BITS;
			u64 reserved	: NVM_12_RESERVED;
		} g;
		/* 2.0 device format */
		struct {
			u64 grp		: NVM_GEN_CH_BITS;
			u64 pu		: NVM_GEN_LUN_BITS;
			u64 chk		: NVM_GEN_BLK_BITS;
			u64 sec		: NVM_20_SEC_BITS;
			u64 reserved	: NVM_20_RESERVED;
		} m;
	
		u64 ppa;
	}
}
{% endhighlight %}

<small>[OCSSD 2.0 문서](http://lightnvm.io/docs/OCSSD-2_0-20180129.pdf)에 기재된 chunk model 은 기존 펌웨어에서 block model 과 비슷하므로 생략한다.</small>

## Physical Page Address Command Set

본 섹션은 Physical Page Address 명령 셋 스펙에 대해 명시하였다. Admin 그리고 I/O 명령 셋에 대한 내용으로 구성되어 있으며 Admin Command 를 통해 디바이스의 geometry 를 알 수 있고 I/O command 를 통해 read, write, erase 명령을 내릴 수 있다. OCSSD 2.0 버전에서는 copy command 도 명시되어 있다. (하지만 리눅스 커널(5.3)에는 관련 OPCODE 를 확인할 수 없다.) 

<small>본 섹션은 OCSSD 2.0 버전 문서에 명시된 내용들이 리눅스 커널에 완전히 반영되지 않아 OCSSD 1.2 버전 중심으로 작성되었다. 본 프로젝트 블로그에서 사용하는 Cosmos+ OCSSD FW 역시 1.2 버전을 기준으로 한다. </small>


### Admin Commands 

| NVMe Opcode  | Command               | 
|:-------------|:----------------------|
| E2h          | [Device Identification](https://skku-vldb-flash.github.io/docs/OCSSD/OCSSD_spec.html#device-identification) | 
| F1h          | Set Bad Block Table   | 
| F2h          | Get Bad Block Table   |


#### Device Identification

Command complete 시에 SSD geometry 정보를 받아온다. 해당 geometry 정보를 기반으로 `nvm_geo` 구조체에 SSD geo 정보를 호스트에서 관리한다. 해당 구조체에 대한 설명은 스펙 문서의 [Device Identification](http://lightnvm.io/docs/Open-ChannelSSDInterfaceSpecification12-final.pdf) 장을 참조하자. 미디어의 타입, 채널/LUN/블록/페이지 개수, 페이지 사이즈 등등의 정보 등이 포함된다.

#### Bad Block Admin Command

Bad Block 관련 admin command 는 2.0 스펙에는 명시되어 있지 않다. 리눅스 5.3 기준으로 pblk target initialize 하는 시점에 get bad block table 명령을 통해 bad block table 정보를 받아온다. 

### IO Commands


| NVMe Opcode  | Command                         | 
|:-------------|:--------------------------------|
| 90h          | Physical Block Erase            |
| 91h          | Physical Page Address Write     |
| 92h          | Physical Page Address Read      | 
| 95h          | Physical Page Address Raw Write (Optional)| 
| 96h          | Physical Page Address Raw Read (Optional)| 

<small>OCSSD 2.0 에는 93h 에 copy 명령도 추가되어 있다.</small>



---

Reference:  
{: .fs-4 }

[OCSSD spec 1.2 (deprecated)](http://lightnvm.io/docs/Open-ChannelSSDInterfaceSpecification12-final.pdf){: .btn .fs-3 .mb-4 .mb-md-0 .mr-3 .mt-3} [OCSSD spec 2.0](http://lightnvm.io/docs/OCSSD-2_0-20180129.pdf){: .btn .btn-purple .fs-3 .mb-4 .mb-md-0 .mr-3 .mt-3}

