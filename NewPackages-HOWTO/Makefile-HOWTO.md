# Makefile-HOWTO

This document will be used to explain what the various sections of a Makefile will have to look like, depending on the OPSI-package type.

The following is the absolute minimum required in the header section:

    # HEADER
    FILES_DIR            = ./files
    BUILD_DIR            = ./build
    OPSI_PACKAGE_NAME    = foobar.op
    OPSI_PACKAGE_VERSION = 1
    SOFTWARE_NAME        = FooBar
    SOFTWARE_VERSION     = 1.0
    TIMESTAMP            = $(shell date -R)
    MAINTAINER_NAME      = John Doe
    MAINTAINER_EMAIL     = maintainer@example.com
    OPSI_PACKAGE_DESC    = This is the $(SOFTWARE_NAME) package, packaged for DotOP by $(MAINTAINER_NAME) on $(TIMESTAMP). It does nothing.

    # Optional entries would go here
    
    TARGET               = $(OPSI_PACKAGE_NAME)_$(SOFTWARE_VERSION)-$(OPSI_PACKAGE_VERSION).opsi


The following is the absolute minimum required in the action section:

    # ACTION
    .PHONY: clean distclean
    
    all: clean build
    
    deploy: all clean install
    
    # Optional actions would expand the "all:" line to e.g. "all: clean someaction build someotheraction"
    
    $(BUILD_DIR)/$(TARGET): ./CLIENT_DATA ./OPSI/control # check that we have the minimum required files/dirs before starting the build
    	mkdir -p $(BUILD_DIR)
    	cp -alf ./CLIENT_DATA $(BUILD_DIR)
    	cp -af ./OPSI $(BUILD_DIR) # no hardlinking here, even though sed would be smart enough to break the link
    	# sed is called to fill the actual values from above into ./build/OPSI/control
    	sed -e "s/OPSI_PACKAGE_NAME/$(OPSI_PACKAGE_NAME)/g" \
    	    -e "s/OPSI_PACKAGE_VERSION/$(OPSI_PACKAGE_VERSION)/g" \
    	    -e "s/OPSI_PACKAGE_DESC/$(OPSI_PACKAGE_DESC)/g" \
    	    -e "s/SOFTWARE_NAME/$(SOFTWARE_NAME)/g" \
    	    -e "s/SOFTWARE_VERSION/$(SOFTWARE_VERSION)/g" \
    	    -e "s/MAINTAINER_NAME/$(MAINTAINER_NAME)/g" \
    	    -e "s/MAINTAINER_EMAIL/$(MAINTAINER_EMAIL)/g" \
    	    -e "s/TIMESTAMP/$(TIMESTAMP)/g" \
    	    -i $(BUILD_DIR)/OPSI/control

    # This is where a "someaction" block would go

    $(TARGET): $(BUILD_DIR)/$(TARGET) # tells make what to do to create the target
    	opsi-makeproductfile $(BUILD_DIR)

    build: $(TARGET) # requires an existing target, will be created according to recipe above if missing

    # This is where a "someotheraction" block would go

    $(TARGET).uploaded:
    	opsi-package-manager -i $(TARGET)
    	touch $(TARGET).uploaded

    install: $(TARGET).uploaded

    clean:
    	rm -rf $(BUILD_DIR)

    distclean: clean
    	rm -rf $(FILES_DIR)
    	rm -f  $(TARGET) $(TARGET).uploaded

Note: All indents in the *ACTION* section must be *TAB* characters, not four spaces, not eight spaces, TABs. This is essential. The make command *will* fail if you do not use TABs.

A Makefile that looks just like that, but without the verbose comments, is stored in the same directory as the file you are reading right now, named Makefile-001-minimum.

To add a download, you must make the following additions to the HEADER section:

    OPSI_PACKAGE_DESC    = This is the $(SOFTWARE_NAME) package, packaged for DotOP by $(MAINTAINER_NAME) on $(TIMESTAMP). It does nothing.
    
    ### BEGIN optional HEADER segment for downloads ###
    DOWNLOAD_URLS        = http://contoso.com/foobar-installer-x86.exe \
                           http://contoso.com/foobar-installer-x64.exe # put multiple URLs on separate lines, end every line except for the last with "\"
    
    DOWNLOAD_FILES       = $(foreach url,$(DOWNLOAD_URLS),$(subst $(dir $(url)),,$(url)))
    DOWNLOAD_TARGETS     = $(addprefix $(FILES_DIR)/,$(DOWNLOAD_FILES))
    #### END optional HEADER segment for downloads ####
    
    TARGET               = $(OPSI_PACKAGE_NAME)_$(SOFTWARE_VERSION)-$(OPSI_PACKAGE_VERSION).opsi

Also, you need to make the following two additions to the ACTION section:

    deploy: all clean install
    
    ### BEGIN optional ACTION segment #1 for downloads ###
    $(DOWNLOAD_TARGETS):
    	mkdir -p $(FILES_DIR)
    	wget $(WGET_COMMON_OPTIONS) -P $(FILES_DIR) $(DOWNLOAD_URLS)
    
    download: $(DOWNLOAD_TARGETS)
    #### END optional ACTION segment #1 for downloads ####
    
    $(BUILD_DIR)/$(TARGET): ./CLIENT_DATA ./OPSI/control
    
    ...
    
    	-i $(BUILD_DIR)/OPSI/control

    ### BEGIN optional ACTION segment #2 for downloads ###
    $(BUILD_DIR)/$(DOWNLOAD_FILES): download
    	mkdir -p $(BUILD_DIR)/CLIENT_DATA/files/
    	cp -alf $(FILES_DIR)/$(subst $(dir $@),,$@) $(BUILD_DIR)/CLIENT_DATA/files/
    #### END optional ACTION segment #2 for downloads ####

    $(TARGET): $(BUILD_DIR)/$(TARGET)

Then, change the line
    $(TARGET): $(BUILD_DIR)/$(TARGET)
to:
    $(TARGET): $(BUILD_DIR)/$(TARGET) $(BUILD_DIR)/$(DOWNLOAD_FILES) # this will trigger the download process

A Makefile that looks just like that, but without the verbose comments, is stored in the same directory as the file you are reading right now, named Makefile-002-download.
