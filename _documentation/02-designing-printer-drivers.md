---
title: Designing Printer Drivers
toc: true
toc_sticky: true
h_range: [1,2]
---
**This document contains a complete tutorial as well as information for manufacturers with examples for designing printer drivers. If you are looking for information regarding the use of printer drivers, kindly refer to <a href="../05-User-Manual/">User Manual</a>**  


<h2 id="introduction"> Introduction </h2>

A driver is a code or data specific to a certain model or group of hardware devices, needed to make the hardware work with the hardware-model-independent code of the operating system. Printing in Linux has moved towards [Driverless Printing](https://openprinting.github.io/driverless/), which means there is no need for any hardware-model-specific code or data. However, there are some problems with the current framework. For example, some printers(especially the old ones) that cannot handle IPP requests are devoid of driverless printing capability. Printer Applications help to address these issues. Kindly refer <a href="../01-printer-application/">Printer Applications - A new way to print in Linux</a> to learn more about Printer Applications, its working and benefits.

For Designing the Printer Application Driver, it would be a lot of re-inventing the wheel if everyone who wants to create a printer driver has to implement all things from scratch. Therefore Michael Sweet has developed <a href="https://github.com/michaelrsweet/pappl/">PAPPL</a>, a library that provides all the common functionality which is required in every Printer Application.

The flowchart below mentions the components of the driver that needs to be designed by you (boxes in blue color), along with the PAPPL utilities (boxes in red color) that would be used by your designed components.

![Untitled Diagram (3)](https://user-images.githubusercontent.com/43112419/93055564-cb98df00-f688-11ea-9cbd-a6b9d40230c1.png)  

The following tutorial helps you to understand how to design each component and integrate the PAPPL Utilities to reduce your work. 

<h2 id="components"> Components for PAPPL-based Printer Driver </h2>

* ### papplMainLoop

    papplMainLoop is the main entry point for the application. It runs a standard main loop with a [system callback](#system_cb()). Also allows the printer application to define its own [usage callback](#usage_cb()) and have an [application specific subcommand](#subcmd_cb()).

        int papplMainloop(
            int                  argc,
            char                 *argv[],
            const char           *version,
            pappl_ml_usage_cb_t  usage_cb,  
            const char           *subcmd_name,
            pappl_ml_subcmd_cb_t subcmd_cb,
            pappl_ml_system_cb_t system_cb,
            void                 *data
        )

    1. ### Version number

        This argument is simply a string literal, denoting the version of the printer driver.
        
        Examples of valid version number are "1.0", "2.1", etc.
    <br>

    2. <h3 id="usage_cb()">  Usage Callback </h3>

        This function helps the user to know about the capabilities of your printer by showing the set of sub-commands and options supported by your printer. For retrieving the output of this function, the user needs to pass the `--help` argument.

        One may specify the usage callback argument as `NULL` in the papplMainloop function. The `default` function is utilized in this situation. The default function shows the default list of sub-commands and options. The output of the default usage callback function is shown below.
            
            Usage: <basename> SUB-COMMAND [OPTIONS] [FILENAME]
                <basename> [OPTIONS] [FILENAME]
                <basename> [OPTIONS] -

            Sub-commands:
            add PRINTER      Add a printer.
            cancel           Cancel one or more jobs.
            default          Set the default printer.
            delete           Delete a printer.
            devices          List devices.
            drivers          List drivers.
            jobs             List jobs.
            modify           Modify a printer.
            options          List printer options.
            printers         List printers.
            server           Run a server.
            shutdown         Shutdown a running server.
            status           Show server/printer/job status.
            submit           Submit a file for printing.

            Options:
            -a               Cancel all jobs (cancel).
            -d PRINTER       Specify printer.
            -j JOB-ID        Specify job ID (cancel).
            -m DRIVER-NAME   Specify driver (add/modify).
            -n COPIES        Specify number of copies (submit).
            -o NAME=VALUE    Specify option (add,modify,server,submit).
            -u URI           Specify ipp: or ipps: printer/server.
            -v DEVICE-URI    Specify socket: or usb: device (add/modify).
        

        If your printer supports only the default set of sub-commands, you can use the default usage callback function. If you wish to output different information when the user uses --help command-line argument, follow the designing **[usage callback function](#usagecallback)** guidelines.


    3. ### Sub-command name

        This argument is a string literal that denotes the sub-command supported by your printer driver. By default, PAPPL helps each driver to support the following sub-commands:

        * add
        * cancel
        * default
        * delete
        * devices
        * drivers
        * jobs
        * modify
        * options
        * printers
        * server
        * shutdown
        * status
        * submit

        If your printer doesn't support any additional sub-command, you may specify the argument as `NULL`.

    4. <h3 id="subcmd_cb()">  Sub-command Callback </h3>

        This callback argument is the function that is to be executed when the sub-command(other than the default sub-commands) is specified.
        
        Since each driver is not bound to support additional sub-command, one may specify this argument as `NULL` in case the printer driver does not support any additional sub-command.

        If you wish to support additional sub-command, then specify the name of the sub-command in the **sub-command name** argument and design a function using the **[design sub-command callback](#subcommandcallback)** guidelines.

    5. <h3 id="system_cb()"> System Callback </h3>

        The System Callback function creates a system object and allows restoring the previous system configuration(if any). Hence, it cannot be `NULL`. Kindly refer to the **[Design system callback](#systemcallback)** guidelines for retrieving more information about the System Callback function.

    6. ### Data

        This argument is a string literal and signifies the context of the printer driver.  

<h2 id="design"> Designing Components </h2>

* <h3 id = "usagecallback"> Usage Callback </h3>

        pappl_ml_usage_cb_t  usage_cb (
            void *data
        );

    The usage callback function receives only one argument, i.e. the callback data.
    
    The function objective is to show the usage details of the printer. Hence, the design guidelines for this function is quite trivial. It consists of a series of `printf` statements describing each supported sub-command and option. It can then return.

___

* <h3 id = "subcommandcallback"> Sub-Command Callback </h3>

    This function allows the printer application to have an application-specific subcommand.

        pappl_ml_subcmd_cb_t subcmd_cb (
            const char      *base_name,
            int             options,
            cups_option_t   *options,
            int             num_files,
            char            **files,
            void            *data
        );

    The Sub-Command callback function receives six arguments Basename, Number of options, Options, Number of files, Name of files, and Callback data. It then returns a new sub-command object.

___

* <h3 id = "systemcallback"> System Callback </h3>
        
        pappl_ml_system_cb_t *system_cb (
            int             num_options,
            cups_option_t   *options,
            void            *data
        );

    The system callback function receives three arguments Number of options, Options, and Callback data. It then returns a new system object.

    **Design Guidelines:**

    1. **Declare Objects and Variables**
    
        The following objects and variables have to be declared:

        | DataType         | Variable Name | Significance         |
        |------------------|---------------|----------------------|
        | pappl_system_t*  | system        | System object        |
        | char*            | val           | Current option value |
        | char*            | hostname      | Hostname             |
        | char*            | logfile       | Log file             |
        | char*            | system_name   | System name          |
        | char*            | spooldir_name | spool-directory      |
        | char*            | authservice   | auth-service         |        
        | pappl_loglevel_t | loglevel      | Log level            |
        | int              | port          | Port number          |
        | pappl_soptions_t | soptions      | System options       |
        | pappl_version_t  | versions      | Software versions    |


        | Note                                                                                                                                       |
        |--------------------------------------------------------------------------------------------------------------------------------------------|
        |Naturally, you can define other names for variables. We have assumed them as given in the table to reduce ambiguity in the subsequent steps.|

    2. **Fetch values using cupsGetOption**

        From the above listed objects/variables, the following variables can get their values from corressponding option name, using `cupsGetOption` PAPPL utility: 

        | Variable Name | Option name     |
        |---------------|-----------------|
        | loglevel      | log-level       |
        | logfile       | log-file        |
        | hostname      | server-hostname |
        | system_name   | system-name     |
        | port          | server-port     | 

        The syntax for using `cupsGetOption` is:

            <variable_name> = cupsGetOption(<option_name>, num_options, options)
        
        Notes:

        * The return type for `cupsGetOption` utility is char*. Hence for "log-level" and "server-port" options, first fetch them into the `val` variable.
        
        * The values taken by `loglevel` variable based on value returned by "log-level" `cupsGetOption` can be summarised using the following table:

            | log-level Option | Value for loglevel variable |
            |------------------|-----------------------------|
            | fatal            | PAPPL_LOGLEVEL_FATAL        |
            | error            | PAPPL_LOGLEVEL_ERROR        |
            | warn             | PAPPL_LOGLEVEL_WARN         |
            | info             | PAPPL_LOGLEVEL_INFO         |
            | debug            | PAPPL_LOGLEVEL_DEBUG        |


        * The `port` variable can be retrived from "server-port" option using [atoi](https://linux.die.net/man/3/atoi) function.

        * Don't forget to add a check for the port number. It must be between 0 to 65535(both inclusive).

    3. **Create a system using papplSystemCreate**

        The variables defined and fetched above are passed to papplSystemCreate utility to create a system.

            pappl_system_t* papplSystemCreate(
                pappl_soptions_t    options,
                const char          *name,
                int                 port,
                const char          *subtypes,
                const char          *spooldir,
                const char          *logfile,
                pappl_loglevel_t    loglevel,
                const char          *auth_service,
                bool                tls_only
            );
        
        * The `soptions` variable is passed as the first argument, which is an bitwise `ORed` of the following options:
        
            |        System Option        |                            Significance                            |
            |:---------------------------:|:------------------------------------------------------------------:|
            | PAPPL_SOPTIONS_DNSSD_HOST   | Use hostname in DNS-SD service names instead of serial number/UUID |
            | PAPPL_SOPTIONS_LOG          | Include link to log file                                           |
            | PAPPL_SOPTIONS_MULTI_QUEUE  | Support multiple printers                                          |
            | PAPPL_SOPTIONS_NETWORK      | Include network settings page                                      |
            | PAPPL_SOPTIONS_NONE         | No options                                                         |
            | PAPPL_SOPTIONS_RAW_SOCKET   | Accept jobs via raw sockets                                        |
            | PAPPL_SOPTIONS_REMOTE_ADMIN | Allow remote queue management (vs. localhost only)                 |
            | PAPPL_SOPTIONS_SECURITY     | Include user/password settings page                                |
            | PAPPL_SOPTIONS_STANDARD     | Include the standard web pages                                     |
            | PAPPL_SOPTIONS_TLS          | Include TLS settings page                                          |
                    
        * The `system_name` variable fetched in 2<sup>nd</sup> Step is passed as second argument. Note that the system-name might be `NULL`. So, don't forget to add a check and pass a default value, in case the fetched system-name is `NULL`.
        * The port number is passed as the third argument. You may pass it as `0` for auto.
        * The 4th argument is a string literal that signifies DNS-SD sub-types. One may pass `NULL` for none.
        * Pass the `spooldir`, `logfile` and `loglevel` variable fetched in 2<sup>nd</sup> Step as fifth, sixth and seventh argument respectively. You can pass `NULL` for the fifth and sixth arguments for default values.
        * The 8th argument signifies the PAM authentication service. You may pass `auth_service` variable fetched in 2<sup>nd</sup> Step or `NULL` for none.
        * If the system only supports TLS connection, pass true in the 9th argument, else false. 


    4. **Add system configurations**

        The system object has tons of configurable attributes and correspondingly a huge number of PAPPL utilities to configure them. These include utilities like Setting Hostname, Setting the footer HTML for the web interface, etc. A detailed list of these function can be found at <a href="../../pappl-system-utilities/">PAPPL System Utilities</a>. 


    5. **Call the [Driver setup function](#setup)**

        Pass the system object created in previous steps as an argument to the Driver setup function. See the [design guidlines](#setup) for knowing about the Driver setup function.
    
    6. **Return System Object**

        The system object created and set up in the previous steps is returned in this step.  

___

* <h3 id = "setup"> Driver setup Function </h3>
    
    This function defines the list of printer drivers and [driver callback function](#callback).

        void pcl_setup (
            pappl_system_t *system
        );
    
    The Driver setup function receives only one argument, i.e. system object.

    **Design Guidelines:**

    1. **Define Drivers name and description**

        Create two arrays of string literals, one for the names of the drivers and the other for their corresponding descriptions. Initialise them suitably and use as directed in 2<sup>nd</sup> Step.

    2. **Call papplSystemSetPrintDrivers**

            void papplSystemSetPrintDrivers(
                pappl_system_t      *system,
                int                 num_names,
                const char * const  *names,
                const char * const  *desc,
                pappl_pdriver_cb_t  cb,
                void                *data
            );

        * The received pappl_system_t object is passed as the first argument.
        * Pass the number of drivers as the second argument
        * The defined driver name and description array are passed as the third and fourth arguments respectively. 
        * Pass the [Callback Function](#callback) as fifth argument.
        * Pass the callback data as the 6th argument. 

___

* <h3 id = "callback"> Callback Function </h3>

    This function tells the printer application what functions to call when performing job functions like starting a job, starting a page, writing a line, ending a page, ending a job, printing a raw job. Driver capability information and defaults(such as resolution, color, etc.) are also provided here.

        bool pcl_callback (
            pappl_system_t          *system,
            const char              *driver_name,
            const char              *device_uri,
            pappl_pdriver_data_t    *driver_data,
            ipp_t                   **driver_attrs,
            void                    *data
        );

    The callback function receives six arguments System object, Driver name, Device URI, Driver data, Driver Attributes, and Callback data. It then returns either `true` on success or `false` on failure.

    **Design Guidelines:**
    
    1. **Add suitable checks**

        You may check that the passed values of `driver_name`, `device_uri`, `driver_data`, and `data` variable is non-NULL. Further, you must verify that the callback data is the same as the `data` variable.

    2. **Assign values to common `driver_data` members**

        All the required information is stored in the `pappl_pdriver_data_t` <a href="#structdriverdata"><sup>[1]</sup></a> structure. These are a few examples of common driver attributes. For the entire list of attributes that can be provided, please look at the `pappl_pdriver_data_t` <a href="#structdriverdata"><sup>[1]</sup></a> structure.

        | Member             | Significance                                         |
        |--------------------|------------------------------------------------------|
        | identify           | [Identify-Printer function](#identify)               |
        | identify_default   | "identify-actions-default" values                    |
        | identify_supported | "identify-actions-supported" values                  |
        | print              | [Print (file) function](#print)                      |
        | rendjob            | [End raster job function](#rendjob)                  |
        | rendpage           | [End raster page function](#rendpage)                |
        | rstartjob          | [Start raster job function](#rstartjob)              |
        | rstartpage         | [Start raster page function](#rstartpage)            |
        | rwrite             | [Write raster line function](#writeline)             |
        | status             | [Status function](#status)                           |
        | format             | Printer-specific format                              |
        | orient_default     | "orientation-requested-default" value                |
        | quality_default    | "print-quality-default" value                        |

    3. **Assign rest of the values `driver_data` members based on `driver_name`**

        Here is the list of all the attributes of `pappl_pdriver_data_t` structure, used to describe driver capability information and defaults. The ones that were not assigned in the previous step may be assigned depending on the name of the driver.


        | Member                        | Significance                                        |
        |-------------------------------|-----------------------------------------------------|
        | bin[PAPPL_MAX_BIN]            | Output bins                                         |
        | bin_default                   | Default output bin                                  |
        | borderless                    | Borderless margins supported?                       |
        | bottom_top                    | Bottom and top margins in hundredths of millimeters |
        | color_default                 | "print-color-mode-default" value                    |
        | content_default               | "print-content-default" value                       |
        | darkness_supported            | printer/print-darkness-supported (0 for none)       |
        | duplex                        | Duplex printing modes supported                     |
        | features[PAPPL_MAX_VENDOR]    | "ipp-features-supported" values                     |
        | finishings                    | "finishings-supported" values                       |
        | force_raster_type             | Force a particular raster type?                     |
        | format                        | Printer-specific format                             |
        | gdither                       | , 'text', and 'graphic' dither array                |
        | icons[3]                      | "printer-icons" values                              |
        | identify                      | Identify-Printer function                           |
        | identify_supported            | "identify-actions-supported" values                 |
        | kind                          | "printer-kind" values                               |
        | make_and_model[128]           | "printer-make-and-model" value                      |
        | media[PAPPL_MAX_MEDIA]        | Supported media                                     |
        | media_ready[PAPPL_MAX_SOURCE] | Ready media                                         |
        | mode_supported                | label-mode-supported                                |
        | num_bin                       | Number of output bins                               |
        | num_features                  | Number of "ipp-features-supported" values           |
        | num_media                     | Number of supported media                           |
        | num_source                    | Number of media sources (trays/rolls)               |
        | num_type                      | Number of media types                               |
        | num_vendor                    | Number of vendor attributes                         |
        | orient_default                | "orientation-requested-default" value               |
        | output_face_up                | Does output media come out face-up?                 |
        | pdither                       | dither array                                        |
        | ppm_color                     | "pages-per-minute-color" value, if any              |
        | print                         | Print (file) function                               |
        | quality_default               | "print-quality-default" value                       |
        | raster_types                  | "pwg-raster-document-type-supported" values         |
        | rendjob                       | End raster job function                             |
        | rendpage                      | End raster page function                            |
        | rstartjob                     | Start raster job function                           |
        | rstartpage                    | Start raster page function                          |
        | rwrite                        | Write raster line function                          |
        | scaling_default               | "print-scaling-default" value                       |
        | sides_default                 | "sides-default" value                               |
        | source[PAPPL_MAX_SOURCE]      | Media sources                                       |
        | speed_default                 | print-speed-default                                 |
        | status                        | Status function                                     |
        | tear_offset_supported[2]      | label-tear-offset-supported (0,0 for none)          |
        | top_offset_supported[2]       | media-top-offset-supported (0,0 for none)           |
        | tracking_supported            | media-tracking-supported                            |
        | type[PAPPL_MAX_TYPE]          | Media types                                         |
        | vendor[PAPPL_MAX_VENDOR]      | Vendor attribute names                              |
        | y_default                     | Default resolution                                  |

    <br>

    **Difference between [print](#print) and [write](#writeline) function**

    Both the print and write functions execute a similar task, but they are invoked by the printer application in a different situation.

    If the printer-specific format, i.e. the format which is understood by the printer and the format of the Job supplied is the same, then the `print` function is invoked. One may rightly guess that this is the case of raw print since no format conversion or processing is required.

    In the other case, the `write` function is used. This is complemented using other functions such as `rstartpage`, `rendpage`, `rstartjob`, and `rendjob`.  

    ![writevsprint (2)](https://user-images.githubusercontent.com/43112419/94988119-a234ea00-0588-11eb-9228-d9c64d86314e.png)

___

* <h3 id = "identify"> Identify Function </h3>
    
     The function helps to identify a printer using display, flash, sound, or speech.

        void pcl_identify(
            pappl_printer_t          *printer,
            pappl_identify_actions_t actions,
            const char               *message
        );

    The Identify function receives three arguments that are the Printer, Actions to take, and Messages (if any).

___

* <h3 id = "print"> Print Function </h3>

    It is used to print a raw job - called if the job format is the same as the format specified by the [driver callback](#callback).

        bool pcl_print(
            pappl_job_t      *job,
            pappl_poptions_t *options,
            pappl_device_t   *device
        );

    The Print function receives three arguments that are Job, Job Options, and device. The function returns true on success and false on failure.

___

* <h3 id = "rendjob"> End Job Function </h3>

    This function is called to end a job.

        bool pcl_rendjob(
            pappl_job_t      *job,
            pappl_poptions_t *options,
            pappl_device_t   *device
        );

    The End Job function receives three arguments that are Job, Job Options, and device. The function returns true on success and false on failure.

    Note that the job data set by [Start Job Function](#rstartjob) must be freed in this function.

___

* <h3 id = "rendpage"> End Page Function </h3>

    This function is called each time a page is completed. It helps in resetting the buffers used by the driver.

        bool pcl_rendpage(
            pappl_job_t      *job,
            pappl_poptions_t *options,
            pappl_device_t   *device,
            unsigned         page
        )
    
    The End Page function receives four arguments that are Job, Job Options, Device, and Page Number. The function returns true on success and false on failure.

___

* <h3 id = "rstartjob"> Start Job Function </h3>

    This function is called when starting a job. The job data is stored with the job.

        bool pcl_rstartjob(
            pappl_job_t      *job,
            pappl_poptions_t *options,
            pappl_device_t   *device
        );

    The Start-Job function, like the End Job Function, receives three arguments that are Job, Job Options, and device. The function returns true on success and false on failure.

___

* <h3 id = "rstartpage"> Start Page Function </h3>

    This function is called when starting a page. Information regarding the page is obtained from the page header and attributes like resolution, margins, page size, orientation, and graphics are set appropriately.

        bool pcl_rstartpage(
            pappl_job_t      *job,
            pappl_poptions_t *options,
            pappl_device_t   *device,
            unsigned         page
        )

    The Start Page function receives four arguments that are Job, Job Options, Device, and Page Number. The function returns true on success and false on failure.

___

* <h3 id = "writeline"> Write Line Function </h3>

    This function writes a line of graphics.

        bool pcl_rwrite(
            pappl_job_t         *job,
            pappl_poptions_t    *options,
            pappl_device_t      *device,
            unsigned            y,
            const unsigned char *pixels
        )

    The Write Line function receives five arguments that are Job, Job Options, Device, Line number, and Line. The function returns true on success and false on failure.

___

* <h3 id = "status"> Printer Status Function </h3>

    This function gets the printer status.

        bool pcl_status(
            pappl_printer_t *printer
        )

    The Printer Status Function receives only one argument and that is the printer. It returns true on success and false on failure.


<h2 id="non-raster"> Add Support for Non-Raster Printers</h2>

Currently, PAPPL supports only raster printers and that too for very few specific input formats like JPEG and PNG. For adding support for non-raster printers like PDF and PostScript printers, you need to supply an external utility that converts the whole job's data into a data stream which the printer understands. Refer to the below-mentioned steps to know how to implement the same.

* **Set the printer-specific format in [callback function](#callback).**

    You need to set `driver_data.format` in [callback function](#callback) to the printer-specific format, i.e. the format/language accepted by the printer. A few examples could be "application/postscript", "application/pdf", etc.

* **Add a filter callback**

    You need to add filter callback from format received by the printer application(formats which you wish your printer can support and accept the job in) to printer-specific format(format/languages that the printer actually understands) using `papplSystemAddMIMEFilter` utility.

        void papplSystemAddMIMEFilter(
            pappl_system_t *system,
            const char *srctype,
            const char *dsttype,
            pappl_mime_filter_cb_t cb,
            void *data
        );

    | Parameter | Significance                                  |
    |-----------|-----------------------------------------------|
    | system    | System                                        |
    | srctype   | Source MIME media type (constant) string      |
    | dsttype   | Destination MIME media type (constant) string |
    | cb        | [Filter callback function](#filtercallback)                       |
    | data      | Filter callback data                          |

    This utility is added in [system callback](#systemcallback) after calling [setup function](#setup) in Step 5.

* <h4 id="filtercallback" style="font-weight: bold;"> Define filter callback function</h4>

    The filter callback function converts the whole job's data into a data stream which the printer understands.

        bool papplJobFilter(
            pappl_job_t    *job,
            pappl_device_t *device,
            void           *data
        )

    The filter callback function receives 3 parameters, that are the Job, the Device, and the Filter data.

    Depending upon the features, stability, ease of use, documentation, pricing, and license you may use any third party API to perform this task of conversion.

    The default filter callback function added in PAPPL are `_papplJobFilterJPEG` and `_papplJobFilterPNG` which can be found in the file [Job-Filter.c](https://github.com/michaelrsweet/pappl/blob/master/pappl/job-filter.c) and can be referenced for writing your own callback function.


<h2 id="template"> Template for PAPPL-based Printer Driver </h2>

    //
    // Include necessary headers
    //
    # include <pappl/pappl.h>


    // Declare structure for Job Data


    // Declare supported media sizes for different models of printers


    //
    // Declare local functions
    //

    static bool   callback(pappl_system_t *system, const char *driver_name, const char *device_uri, pappl_pdriver_data_t *driver_data, ipp_t **driver_attrs, void *data);
    static void   compress_data(pappl_job_t *job, pappl_device_t *device, unsigned char *line, unsigned length, unsigned plane, unsigned type);
    static void   identify(pappl_printer_t *printer, pappl_identify_actions_t actions, const char *message);
    static bool   print(pappl_job_t *job, pappl_poptions_t *options, pappl_device_t *device);
    static bool   rendjob(pappl_job_t *job, pappl_poptions_t *options, pappl_device_t *device);
    static bool   rendpage(pappl_job_t *job, pappl_poptions_t *options, pappl_device_t *device, unsigned page);
    static bool   rstartjob(pappl_job_t *job, pappl_poptions_t *options, pappl_device_t *device);
    static bool   rstartpage(pappl_job_t *job, pappl_poptions_t *options, pappl_device_t *device, unsigned page);
    static bool   rwrite(pappl_job_t *job, pappl_poptions_t *options, pappl_device_t *device, unsigned y, const unsigned char *pixels);
    static void   setup(pappl_system_t *system);
    static bool   status(pappl_printer_t *printer);
    static pappl_system_t   *system_cb(int num_options, cups_option_t *options, void *data);


    //
    // 'main()' - Main entry for the hp-printer-app.
    //

    int
    main(int  argc,             // I - Number of command-line arguments
        char *argv[])           // I - Command-line arguments
    {
    papplMainloop(argc, argv, "1.0", NULL, NULL, NULL, system_cb, "hp_printer_app");
    return (0);
    }


    //
    // 'callback()' - PCL callback.
    //

    static bool                // O - `true` on success, `false` on failure
    callback(
        pappl_system_t       *system,      // I - System
        const char           *driver_name,   // I - Driver name
        const char           *device_uri,      // I - Device URI
        pappl_pdriver_data_t *driver_data,   // O - Driver data
        ipp_t                **driver_attrs, // O - Driver attributes
        void                 *data)    // I - Callback data
    {

    }


    //
    // 'compress_data()' - Compress a line of graphics.
    //

    static void
    compress_data(
        pappl_job_t    *job,        // I - Job object
        pappl_device_t *device,     // I - Device
        unsigned char  *line,       // I - Data to compress
        unsigned       length,      // I - Number of bytes
        unsigned       plane,       // I - Color plane
        unsigned       type)        // I - Type of compression
    {
    
    }


    //
    // 'identify()' - Identify the printer.
    //

    static void
    identify(
        pappl_printer_t          *printer,  // I - Printer
        pappl_identify_actions_t actions,   // I - Actions to take
        const char               *message)  // I - Message, if any
    {
    // Identify a printer using display, flash, sound, or speech.
    }


    //
    // 'print()' - Print file.
    //

    static bool                           // O - `true` on success, `false` on failure
    print(
        pappl_job_t      *job,            // I - Job
        pappl_poptions_t *options,        // I - Options
        pappl_device_t   *device)         // I - Device
    {

    }


    //
    // 'rendjob()' - End a job.
    //

    static bool                     // O - `true` on success, `false` on failure
    rendjob(
        pappl_job_t      *job,      // I - Job
        pappl_poptions_t *options,  // I - Options
        pappl_device_t   *device)   // I - Device
    {

    }


    //
    // 'rendpage()' - End a page.
    //

    static bool                     // O - `true` on success, `false` on failure
    rendpage(
        pappl_job_t      *job,      // I - Job
        pappl_poptions_t *options,  // I - Job options
        pappl_device_t   *device,   // I - Device
        unsigned         page)      // I - Page number
    {

    }


    //
    // 'rstartjob()' - Start a job.
    //

    static bool                     // O - `true` on success, `false` on failure
    rstartjob(
        pappl_job_t      *job,      // I - Job
        pappl_poptions_t *options,  // I - Job options
        pappl_device_t   *device)   // I - Device
    {

    }


    //
    // 'rstartpage()' - Start a page.
    //

    static bool                      // O - `true` on success, `false` on failure
    rstartpage(
        pappl_job_t       *job,       // I - Job
        pappl_poptions_t  *options,   // I - Job options
        pappl_device_t    *device,    // I - Device
        unsigned          page)       // I - Page number
    {

    }


    //
    // 'rwrite()' - Write a line.
    //

    static bool             // O - `true` on success, `false` on failure
    rwrite(
        pappl_job_t         *job,       // I - Job
        pappl_poptions_t    *options,   // I - Job options
        pappl_device_t      *device,    // I - Device
        unsigned            y,      // I - Line number
        const unsigned char *pixels)    // I - Line
    {

    }


    //
    // 'setup()' - Setup PCL drivers.
    //

    static void
    setup(
        pappl_system_t *system)      // I - System
    {
    
    }


    //
    // 'status()' - Get printer status.
    //

    static bool                   // O - `true` on success, `false` on failure
    status(
        pappl_printer_t *printer) // I - Printer
    {

    }


    //
    // 'system_cb()' - System callback.
    //

    pappl_system_t *            // O - New system object
    system_cb(int           num_options,    // I - Number of options
        cups_option_t *options, // I - Options
        void          *data)        // I - Callback data
    {

    }

<h2 id="guidelines"> Design Guidelines </h2>

* **1 Printer/Scanner Application = 1 Snap:**
Don't make a Snap which contains tons of different printer and scanner applications. This is to ensure that you don't occupy lots of ports and network resources when you have many printer and scanner applications.

* **Printer/Scanner/Fax support can be in a single application:**
To support multi-function devices you can put all the printer, scanner, and fax support into one application.
For example, we recommend [HPLIP](https://developers.hp.com/hp-linux-imaging-and-printing) to be put into one single application.  

* **Recommended: 1 Printer/Scanner Application per project or manufacturer/product line:**
It may be possible that a multi-function device may be served by different applications for printing and scanning. For example for legacy devices, scanning may be supported by retrofitted sane-airscan scanner application while printing is supported by the Gutenprint printer application. But here it is easier and better for the project, organization, and management that each project has its own printer/scanner application snap. So SANE will maintain a SANE scanner application snap and they put it on the Snap Store. Similarly, Gutenprint, HPLIP, foo2zjs, Epson, Canon, Ricoh, and so on will do the same.

* **NOT make 1 Printer/Scanner Application for each device:** 
For example, HP Laserjet 4 and HP Laserjet 5 should not have different applications. Otherwise, Snap Store will be cluttered with thousands of applications, and spotting the real application would be difficult. Also, it would result in a lot of code duplication, requiring more storage on the user's machine.

* **1 Printer/Scanner Application = 1 Port:**
If there are several devices connected to the system and they are served by one printer/scanner application, do not open ports for each device. Because you may run out of ports. Also, ports are not always the same on each boot as other applications may start up before on the next boot.

* **For more than 1 device on 1 Application use URI: ipp://localhost:\<PORT\>/ipp/print/\<NAME\>**
This is the recommended way to cope up with several devices by only using a single port.

* **DNS-SD service names must be always the same:**
They must be independent of order application start at boot or of device discovery. To make sure that a printer application can serve several devices of the same model include the DNS-SD service name the CEON number of the devices.

* **Web admin interface should allow:**
    1. **suppressing auto-setup for selected devices:**
    Auto detecting devices might be unsuitable for some cases. For example, if two printer applications support the same device, the user must be able to select with which application he wishes to print. So web interface must contain the option to somehow blacklist a printer application.

    2. **manual setup of additional devices/instances**
    Necessary for legacy printers that cannot be auto-discovered in the network. 

    3. **configuration of options not accessible via IPP**
    Many manufacturers have options that cannot be translated into IPP attributes. So web interface has to provide the possibility to set up these options.

* **sane-airscan in SANE Scanner Application must be built without IPP Scan to avoid recursive discovery infinite loop (“Scanner bomb”)**

<h2 id="resources"> Resources </h2>

[1] <a href="../01-printer-application/">Printer Application</a>
<br>
[2] <a href="https://github.com/michaelrsweet/hp-printer-app">HP Printer App Example</a>
<br>
[3] <a href="https://github.com/michaelrsweet/pappl/">PAPPL</a>
<br>
[4] <a href="../../pappl-system-utilities/">PAPPL System Utilities</a>
<br>
[5] <a href="../04-packaging-drivers/">Packaging Drivers and Uploading them to Snap Store</a>
<br>
[6] <a href="../05-User-Manual/">User Manual</a>
<br>
[7] <a href="https://github.com/OpenPrinting/ps-printer-app">PS Printer App Example</a>