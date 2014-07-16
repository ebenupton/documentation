# Background

## What is MMAL ?
* Stands for “Multi-Media Abstraction Layer”
* A generic API to drive multimedia components (e.g. camera, codecs, renderers, etc)

## Design considerations
* As simple as possible to use from client code
* Allows efficient use of HW acceleration
* Sufficiently generic to support different kinds of multimedia components
* As simple as possible to implement components
* Supports concurrency
* Portability

# MMAL API (Client side API)

## Design goals

The MMAL API has been designed to be:
* As simple to use as possible
  – We really wanted to make client code as simple as possible to write.
  – Even if that meant moving some of the complexity into the MMAL framework itself.
* But still be generic enough
  – To support any kind of multimedia component
  – Be platform agnostic (to support ARM as well as VideoCore components)
* And flexible enough
  – To get the best out of VideoCore

## Design decisions
* API is mostly synchronous
– Asynchronous APIs are more complex to use
– We only use asynchronous calls where it matters for performance reasons
* API exposes a lot of information directly to the client via client visible context structures.
– To make it very easy for the client to access most of the information it needs
– To avoid the client having to duplicate all this information
– To make debugging easier
* API is portable and extensible
– The API itself is self-contained (no dependencies on other components)
– Can be extended without breaking source or binary compatibility
General concepts
MMAL is based on similar concepts to OpenMAX IL:
* Components
– Individual blocks of functionality (e.g. video decoder)
– Client will need to instantiate a component in order to do anything at all
– Components expose a list of input and output ports
– Components also expose a control port
* Ports
– Expose a data stream into or out of the component
– They also contain a full description of the data stream itself
– Most of the MMAL API is about doing something with a port
* Buffer headers
– Data is sent to / received from a port using buffer headers
– Buffer headers are a way to package data with necessary extra metadata
* E.g. data size, buffer size, timestamps, misc flags
– They are also used to help with buffer management
* E.g. managing the lifetime of a buffer or buffer sharing
API
* The MMAL API is doxygen documented. This is probably the best place to start for an exhaustive view of the API.

* interface/mmal/*.h define the public API
– Everything in subdirectories is not part of the public API
– Subdirectories contain internal interfaces
– The util directory is the exception
* interface/mmal/util/*.h
– Collection of helper libraries
– These are also public interfaces

* Most API calls return a status code (see MMAL_STATUS_T in mmal_types.h)
* Most API calls take either a component or a port as an argument
– Depending on whether the action is port specific or applies to the whole component
Components
* Defined in interface/mmal/mmal_components.h
* mmal_component_create()
– Instantiates a component given a string (name which uniquely identifies the component)
– Returns a pointer to a MMAL_COMPONENT_T structure containing:
* a list of input ports
* a list of output ports
* a control port
* mmal_component_destroy()
– Does just the opposite
* mmal_component_enable()/disable()
– A component only processes data on its ports when it is enabled
Ports
* Defined in interface/mmal/mmal_ports.h
* Components expose MMAL_PORT_T structures for all their input, output and control ports.
* A port structure contains a lot of information defining the data stream going through that port:
– Port type (i.e. input, output or control)
– Format of the elementary data stream (MMAL_ES_FORMAT_T)
– Buffer requirements of the port

Ports (cont’d)
* mmal_port_format_commit()
– Allows the client to configure the format of a port
– Changes made by the client to the port format only take effect after a call to commit is done
– A commit on an input port of a component is susceptible to change the format of all the output ports of that component (i.e. their MMAL_PORT_T might have been updated after a call to commit)
* mmal_port_enable()
– A port will only start accepting buffer headers once enabled
– The client needs to provide a callback used to return buffer headers
* mmal_port_send_buffer()
– Used to send buffer headers to a port:
* For input ports: buffers to be emptied
* For output ports: empty buffers to be filled in 
– This is a non-blocking call. Once processed by the port, the buffer header will be returned to the client using the callback setup in mmal_port_enable()
* mmal_port_disable()
– A port will return all unused buffer headers before the call completes

Port buffer requirements
* Ports need a certain number of buffers to be able to function properly
* They also need these buffers to be of a certain size
* All these buffer requirements are described in the port structure
– buffer_num_min : mininum number of buffers required by port
– buffer_size_min: minimum size of buffers required by port
– There are also “recommended” values which represent what a component would ideally want to behave in the most efficient way
* When a port is enabled, the client needs to tell the component which values it actually selected
– Done by setting buffer_num and buffer_size
– buffer_num: maximum number of buffers that can be queued in the component at once
– buffer_size: maximum size of a buffer that can be queued in the component
– It is the responsibility of the client to set these values before enabling a port
Port Format
* Defined in interface/mmal/mmal_format.h
* Describes the format / properties of the elementary stream of data going through a port:
– Type of the elementary stream (i.e. video/audio/subpicture/control)
– A Four Character Code uniquely identifying the encoding of the data stream (e.g. H264). See mmal_encodings.h
– Type specific description of the stream:
* for video: width, height, cropping region, frame rate, aspect ratio
* for audio: channels, samplerate, etc
– Extra codec specific information
* E.g. H264 SPS/PPS
* Format of this data will be documented at some point
* This is basically all the information that doesn’t quite fit in the generic fields above
* The format should contain all the data necessary to identify and process an elementary stream
Buffer Headers
* Defined in interface/mmal/mmal_buffer.h
* They package all the information necessary to describe the actual data payload (e.g. pointer to payload, size of payload, size of payload buffer)
– Centralising this information makes it easier to send buffers around the system
* They also contain generic metadata on the payload
– E.g. timestamps, various flags
* They are also used to help with buffer management
– They are reference counted so that they are only recycled when nothing is using that payload anymore
– Reference is increased by mmal_buffer_header_acquire()
– Reference is released by mmal_buffer_header_release()
– Recycling happens automatically (thanks to a callback setup when the buffer header is created)
Queues (of buffer headers)
* Defined in interface/mmal/mmal_queue.h
* Helper library used to create and manage a queue of MMAL buffer headers
* Often used by clients and components to queue buffer headers if they need to process them asynchronously

Pools (of buffer headers)
* Defined in interface/mmal/mmal_pool.h
* Helper library to create and manage a fixed number of buffer headers
* Doesn’t strictly have to be used by clients
* However this does take in charge a lot of the buffer management that the client would otherwise have to do
* Pools can be created with or without payloads associated with the buffer headers
– Allows the client to do its own memory allocation
* Pools can be resized (both the number of buffer headers and the associated payloads)
* Pools allow clients to set a callback for when buffer headers are recycled back into the pool

Example code
See example_basic_1.c (interface/mmal/test/examples/)
Parameters
* Defined in interface/mmal/mmal_parameters*.h
* Parameters are used as a way to configure the behaviour of a component or port (e.g. which exposure mode should a camera use).
* Parameters are component specific. It is the responsibility of the client to know which parameter is relevant for which component.
* Parameters are uniquely identified by an integer
– We maintain a list of standard parameters
– Customers can also add their own
* Each parameter also has an associated data structure (the data the client / component is interested in)
– This data structure always starts with a predefined header (containing the id and size)
* mmal_port_parameter_set()/get()
– Take a port as a parameter
– Parameters which apply to the whole component must be get / set on the control port of the component
Parameters (cont’d)
* Helper functions are available to get / set parameters which are one these predefined generic types:
– MMAL_PARAMETER_UINT32_T
– MMAL_PARAMETER_INT32_T
– MMAL_PARAMETER_RATIONAL_T
– MMAL_PARAMETER_BOOLEAN_T
* See interface/mmal/utils/mmal_util_params.h
Events
* Since the MMAL API is partly asynchronous (data transfer part), there is also a need for the component to be able to asynchronously signal ‘events’
* Events are for instance needed by the component to signal potential errors while processing data
* Events are very much component specific but there are a few predefined generic ones:
– MMAL_EVENT_ERROR
– MMAL_EVENT_EOS
* The end of stream is generally transmitted in-band with the data (in the buffer header) except when a component doesn’t have an output port
– MMAL_EVENT_FORMAT_CHANGED
* The format of a port has changed. The event itself carries the new format.
* Events are in fact transmitted as MMAL buffer headers on the same ports as the data
– This simplifies a lot of the logic in handling events (in client and framework)
– This allows events to be sent in-band with the data
* Useful if the event needs to be associated with a specific point in the stream
Events (cont’d)
* However they originate from the component
* The control port of a component is also used to receive events which are not associated with any input / output port
– E.g. generic errors
– EOS
Example code
See example_basic_2.c 
Tunnelling
* The API allows 2 ports to be connected directly together
* This hides all the data traffic between the 2 ports from the client
* The ports are also responsible for allocating their own buffers in that case
* This make client code simpler if it doesn’t need to see the data

* Tunnelling is configured by calling mal_port_connect()
– However it is still the responsibility of the client to configure the format of the 2 ports so that they talk the same language
– The output port still needs to be enabled subequently to start the data transfer between the 2 ports
Acquiring / Releasing components
* Components are reference counted
– Reference acquired with mmal_component_acquire()
– Reference released with mmal_component_release()
* When all references are released, the component is automatically destroyed
* mmal_component_destroy() is in fact an alias of mmal_component_release()
* Components are created with an initial reference of 1
* More on its uses later

Payload allocators
* Port can expose memory allocators
– They might have a preference on where memory is allocated (e.g. shared memory areas to avoid copies)
* Clients can use
– mmal_port_payload_alloc() to allocate memory for a port
– mmal_port_payload_free() to free that memory
– mmal_port_pool_create() to allocate pools which will use these memory allocators
* It is recommended that client always use mmal_port_pool_create() when creating a pool as this will ensure that the best memory allocator will be used
* Payload allocators will acquire a reference on the component to make sure that this one won’t be destroyed until all the payloads have been freed
* This does mean that the order in which things are destroyed is not really important
Buffer header replication
* This is a way to duplicate a buffer header without duplicating its content
* This is useful because buffer headers can only be sent to one component at once (they contain a state)
* To send a payload to several components at once, its buffer header needs to be replicated using mmal_buffer_header_replicate()
* However, an easier way to do that might be to use the splitter component (uses replication internally)

* Replicating a buffer header will increment the reference count on the original buffer header
* The reference count is decremented when the replicated buffer header is released (or more precisely recycled)
High Level APIs
(i.e. utility libraries)
Why do we need higher level APIs ?
* The MMAL API was designed on purpose to be relatively low level so as to provide clients with enough control and flexibility.
* Even though we tried to make the MMAL API as simple to use as we could, this does mean that clients wanting to do some very simple things (like connecting MMAL components together) still have to implement a fair amount of code.
* To make things simpler for clients we provide higher level libraries which implement some commonly used features / scenarios.




* Disclaimer: most of these APIs are still experimental and their interfaces might change
MMAL Connections
* interface/mmal/utils/mmal_connection.h
* Provides an interface to connect and manage the connection of 2 MMAL ports
– Still allows the client access to the data by means of a callback
– Automatically sets the port of the input format to match the output one
– Automatically creates and manages buffer headers
– Provides some support for handling the MMAL_EVENT_FORMAT_CHANGED event
– Allows you to use tunnelling (in which case the data won’t be accessible)

* See example code (example_connections.c)
MMAL Graphs
* interface/mmal/utils/mmal_graph.h
* Provides an interface to construct a graph of connected components
* All internal connections to the graph are managed transparently by the graph itself
* Only the external ports of the graph (if any) need to be managed by the client
* It is also possible to create a MMAL component from an existing graph of components

* See example code (example_graph.c)
MMAL Component Wrapper
* interface/mmal/utils/mmal_component_wrapper.h 
* Provides a simplified and fully synchronous API around the MMAL API
* Is only useful for clients wanting to manipulate components in isolation
Framework Architecture
Architecture
MMAL Core
* Is the part of MMAL exposing the public API
– We have a single point of entry into the framework
* Implements the necessary locking to make MMAL thread safe
* Implements a lot of support code for components
– The aim is to make components as simple as possible
* Provides a default implementation for some parts of the API
– Again to make components as simple as possible
* Talks to components using the MMAL component interface
* Manages the registry of components
* Code in interface/mmal/core
MMAL Core – Thread safety
* Client API is thread-safe
* MMAL Core implements the necessary locking
* Locking is done on a per port granularity
– Prevents calls for one port impacting on other ports
* We also use a different (per-port) lock for mmal_port_send_buffer()
– Avoids having control calls block data transfer
MMAL Core – Component Registry
* The core keeps a registry of available components
* Components need to be registered by calling mmal_component_supplier_register()
* Can be done automatically by declaring a MMAL_CONSTRUCTOR() in the component
– Currently relies on a gcc extension so only works on host
* mmal_component_supplier_register()
– Registers a component by supplying a function pointer which will be used to create the component
– Registers a name space for the component by providing a “prefix” string
* Component names are in the form “prefix.name”
* When a client creates a component, the core will use the “prefix” to decide which component to instantiate
MMAL Core - Tunnelling
* Tunnelling is just a way to hide from the client all the data transfer that’s happening between 2 ports
* Tunnelling is implemented mostly at the core level
– Components mostly do not need to know anything about tunnelling
– They just keep receiving and sending data the same way
* When 2 ports are connected together (i.e. tunnelled)
– The core is in charge of creating the buffer headers needed by the ports
– The core is in charge of transferring buffer headers from 1 port to the other
* Components can however override this by providing their own tunnelling functions.
MMAL Core - Logging
* The MMAL core provides some logging facilities
– interface/mmal/logging.h
– available to the core itself and components
* Logging is redirected to vcos logging
* All the calls to the MMAL API are also logged to help debugging
Components
MMAL Components
* Implement the MMAL component interface
* Only called directly by the core
* Need to be registered with the core
* Located in interface/mmal/components

MMAL Component Interface
* The component interface is very similar to the MMAL API
* Interface defined in interface/mmal/core/*.h
Example component – Null sink
* See interface/mmal/components/null_sink.c
* Probably the simplest MMAL component there is
Actions
* The MMAL core provides a way for components to register “actions”
* They are processing functions which will be run by the core in a separate thread
* Actions need to be triggered in order for them to run
– E.g. when a buffer header is received by a component
* The aim is to remove a lot of boiler plate code out of the components themselves
List of existing components - Generic
* null_sink: just discards any data it receives
* copy: copies data it receives on its input port to its output port
* passthrough: passes data from the input port to the output port without any actual copy (sharing the payload by using buffer header replication)
* splitter: similar to passthrough expect that the input data is passed to several output ports
* aggregator: a component which aggregates several other components (using the mmal graph API)
* camera_info: pseudo component which is used to expose information on the cameras available in the system
* container_reader: extracts elementary streams from multimedia files using the container API
* container_writer: writes multimedia files using the container API

List of components – VideoCore only
* video_converter: colour conversion using vc_image conversion functions
* Components making use of the mmal to RIL adaptation layer
– ril.video_decode
– ril.video_encode
– ril.video_render
– ril.image_decode
– ril.image_encode
– ril.camera
– ril.resize
* mmal_vc_api: host side component responsible for the IPC when using VideoCore components
List of existing components - Testing
* artificial_camera: test component used on PC to emulate a camera component
* avcodec_video_decoder: video decoder component used as a test component on PC
* sdl_video_render: video renderer component used as a test component on PC

Architecture – The Full Picture
Architecture
Architecture – The Full Picture
* We have 2 MMAL frameworks running
– One on the host
– One on VideoCore
* We have one component on the host side responsible for proxying components which reside on the VideoCore side
* We have a server on the VideoCore side responsible for forwarding requests from the proxy to the MMAL framework running on VideoCore
* We have a component on the VideoCore side which acts as an adaptation layer for our existing RIL components
Benefits
* We have the same architecture (and same code) running on both sides
* We can easily migrate (some of the) components between the two processors
* We can easily support VideoCore side tunnelling (to avoid some of the IPC overhead)
* We also benefits from all the other features provided by the MMAL core on the VideoCore side
* We can also support VideoCore self-hosted MMAL clients
ARM / VideoCore IPC
IPC implemenation
* We use VCHIQ for the actual communication between the ARM and VideoCore
* Two memory models are supported for transferring data across between the two processor
–  VCHIQ bulk transfers
* With an optimisation where small payloads can be transferred directly into the VCHIQ control message
* To improve concurrency, we also do not wait for the bulk transfer to be completed
* This is the default model
– Using memory shared between the two processor
* The buffer needs to have be allocated using a special shared memory driver
* Shared memory can however be quite expansive to access on some plaftforms
* Shared memory support is mostly transparent for the client
– Need to be switched on per port by using the MMAL_PARAMETER_ZERO_COPY parameter
–  Buffer payloads need to be allocated using the port’s allocator functions
* Or more simply by using mmal_port_pool_create() which just does the right thing
MMAL Proxy Component
* interface/mmal/vc/mmal_vc_api.c
* Responsible for acting as a real component
* Mainly just calls into a simpler IPC API (mmal_vc_msgs.h) to communicate with VideoCore side
* Implements a payload allocator (mmal_port_payload_alloc()) on its ports to provide shared memory support

MMAL Server
* VideoCore side of the IPC layer
* Translates requests from MMAL proxy components into MMAL API calls and vice versa
* It really is just a special MMAL client
– It is responsible for creating and managing buffer header that will be sent to a port
– It is responsible for allocating the buffer header payload when not in shared memory mode
RIL adapation layer
RIL adaptation layer
* RIL is our existing low-level interface for multimedia component
* RIL stands for Reduced OpenMAX IL
* We wanted to reuse all this code
– It is a lot of code that we would have to re-implement otherwise
– It is also already extensively tested
* We provide a MMAL / RIL adaptation layer
* The adaptation is very straight-forward
– The MMAL API is very similar to the RIL interface
* Code in interface/mmal/vc/ril/*

MMAL Opaque support
* RIL components are optimised for “proprietary communication”
– They are only really efficient when they are connected together and talk to each other in a proprietary way.
– For video components, this mostly involves passing vc images directly to each other (usually in striped YUV mode).
* They are less efficient when used in the standard OpenMAX IL way
– Data usually needs to be converted to something that’s standardised by the spec (e.g. I420 images)… to be converted back by the next component.
– Even when no conversion is needed, the vc image can’t be passed as is but needs to be copied into an OMX buffer.
* The MMAL opaque support allows us to expose this proprietary communication to the client
– We define a new MMAL encoding type (MMAL_ENCODING_OPAQUE) which only our components understand
– When in opaque mode, the payload transmitted by the MMAL buffer headers are in fact just a reference to a vc image
Android Integration
Which MM features do we accelerate ?
* Video playback
– Video decoder
– Rendering
* Video recording
– Camera
– Video encoder
– Rendering
* Still picture camera
– Camera
– JPEG encoder
– Rendering
* JPEG decoding
– JPEG decoder
Video decoder / encoder
* Android uses OpenMAX IL as the interface for its codecs
– So we potentially could use our existing OMX IL components
* However it doesn’t quite use OpenMAX IL in a standard way
– E.g. it makes assumptions that the port numbers start from 0
– Which means that we would need an adaptation / integration layer to reuse our existing OMX IL
* It also relies on proprietary extensions
– They need to be supported to provide the best performance and all the features supported by Android
– They are very Android specific and it wouldn’t make sense to integrate that directly into our OMX IL components
* Our OMX IL IPC (ILCS) is not very efficient (compared to the MMAL IPC)
– MMAL supports shared memory (between ARM / VideoCore)
– MMAL uses VCHIQ more efficiently (ILCS is too synchronous)
* We provide a small Android specific OMX IL to MMAL adaptation layer
– We needed an adaptation layer anyway
– We benefit from the advantages that MMAL provides
– Mapping OMX IL to MMAL is straight-forward

Rendering
* Android provides an overlay API used to display video
* Originally implemented using the MMAL video renderer component
* Now deprecated in favor of using WFC directly
Camera
* Android provides its own camera HAL
* Implemented using the MMAL camera component
* With some help from other components
– video_converter (for color conversion)
– image_encoder (for jpeg encoding of still pictures)
JPEG decoding
* All JPEG decoding in Android goes via the Skia Graphics Engine
– Used by web browser, picture viewer, thumbnails, media scanner
* Skia expects an RGBA 8888 output
* HW decoding uses 2 MMAL components
– image_decoder to decode the JPEG data into I420
– video_converter to convert I420 into RGB8888
* We use the MMAL aggregator to package this chain of 2 components
– Simpler client code since there is only 1 component to drive
– More efficient since this reduces the number of IPC (the aggregator runs on VideoCore)
* We use the MMAL “component wrapper” utility library
– Provides an entirely synchronous API to drive the component
– Makes the client code simpler
JPEG decoding (cont’d)
* Problem with decoding JPEG in HW is the setup cost of the decoder
– Create and setting up the MMAL component(s) takes in the order of 20-30ms
– You do not really want to do that per frame you decode
* We work around this by spawning a decoder thread which can be reused for more than 1 decoding
New MMAL features & Work in progress
Clock support
* Some kind of clock support is necessary to provide a fully MMAL based multimedia framework with audio / video synchronisation
* Clock support in OpenMAX IL has some major problems
– An external clock component  is needed to serve all scheduling requests
* This makes scheduling complicated for components
* This adds quite a lot of overhead and latency, especially if the clock is running remotely
– The clock  ports of have to be tunnelled with each other (a clock port needs to get/set parameters on the connected port)
* This prevents from running the clock remotely (can be a problem with an ARM / VideoCore split)
* MMAL works in a very similar way but solves these issues by having each clock port provide its own clock
– Scheduling requests are made to this clock (the request does not cross the component boundary)
– Clock ports are connected to each other only to allow all the different clocks to synchronise themselves
* All the clock support is implemented in the framework itself

Remoting API
* Still work in progress and not yet on vc4/DEV
* Mainly a rewrite of the MMAL ARM/VC IPC code with the aim of making the remoting interface a public interface
* Will also provide a simple remoting I/O interface to allow easily replacing the actual IPC mechanism
* There is some very experimental TCP/IP remoting support which for instance allows client code or some of the MMAL components to run on a PC while the rest execute on target

