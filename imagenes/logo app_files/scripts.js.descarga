"use strict";

angular.module("siteGeneratorStaticsMetadataAppInternal", [ "siteGeneratorStaticsMetadataPreload", "wixAngular" ]);

angular.module("siteGeneratorStaticsMetadataApp", [ "siteGeneratorStaticsMetadataAppInternal" ]).config(function() {
    return;
});

"use strict";

angular.module("siteGeneratorStaticsMetadataApp").factory("BackendLayoutPreset", [ "metadataReader", function(metadataReader) {
    function isSingle(forContent) {
        var singles = [ "Bookings", "GetSubscribers", "Stores", "Blog", "Music", "ProGallery", "BandsInTown" ];
        return singles.indexOf(forContent.split("_")[0]) !== -1;
    }
    function fetchPresets(forContent) {
        return metadataReader.readZip("LayoutPresets/" + forContent).then(function(zip) {
            return JSON.parse(zip.file("data").asText());
        }).catch(function() {
            return [];
        });
    }
    return {
        search: function(options) {
            return fetchPresets("Presets").then(function(presets) {
                if (options && options.forContent && isSingle(options.forContent)) {
                    var filteredPresets_1 = presets.filter(function(preset) {
                        return preset.data.forContent.indexOf(options.forContent) !== -1;
                    });
                    return fetchPresets(options.forContent).then(function(presets) {
                        return presets.concat(filteredPresets_1);
                    });
                } else {
                    return presets;
                }
            });
        }
    };
} ]).factory("BackendLayoutPresetNew", [ "metadataReader", function(metadataReader) {
    function isSingle(forContent) {
        var singles = [ "Bookings", "GetSubscribers", "Stores", "Blog", "Music", "ProGallery", "BandsInTown" ];
        return singles.indexOf(forContent.split("_")[0]) !== -1;
    }
    function fetchPresets(forContent) {
        return metadataReader.readZip("LayoutPresets/" + forContent).then(function(zip) {
            return JSON.parse(zip.file("data").asText());
        }).catch(function() {
            return [];
        });
    }
    return {
        search: function(options) {
            if (options && options.forContent && isSingle(options.forContent)) {
                return fetchPresets(options.forContent);
            } else {
                return fetchPresets("Presets");
            }
        }
    };
} ]);

"use strict";

angular.module("siteGeneratorStaticsMetadataApp").factory("StaticDataHandler", [ "$q", function($q) {
    function StaticDataHandler(data) {
        this.data = data;
    }
    StaticDataHandler.prototype.search = function() {
        return $q.when(this.data);
    };
    return StaticDataHandler;
} ]);

"use strict";

angular.module("siteGeneratorStaticsMetadataApp").factory("BackendTheme", [ "metadataReader", function(metadataReader) {
    function fetchThemes() {
        return metadataReader.readZip("Themes/Themes").then(function(zip) {
            return JSON.parse(zip.file("data").asText());
        }).catch(function() {
            return [];
        });
    }
    return {
        search: function(options) {
            return fetchThemes();
        }
    };
} ]);

"use strict";

try {
    angular.module("siteGeneratorStaticsMetadataPreload");
} catch (e) {
    angular.module("siteGeneratorStaticsMetadataPreload", []);
}

angular.module("siteGeneratorStaticsMetadataPreload").run([ "$templateCache", function($templateCache) {
    "use strict";
    $templateCache.put("views/site-generator-statics-metadata-app.html", "<div class='container' ng-click='siteGeneratorStaticsMetadataApp.onClick()'>\n" + "<div class='hero-unit'>\n" + "<h1>\n" + "{{'general.YO' | translate}}\n" + "<i class='logo-on-header site-generator-statics-metadata-svg-font-icons-wix-logo'></i>\n" + "</h1>\n" + "<div>\n" + "This is {{siteGeneratorStaticsMetadataApp.name}} ({{siteGeneratorStaticsMetadataApp.clicks}})\n" + "</div>\n" + "<h3>Enjoy coding! - Yeoman</h3>\n" + "</div>\n" + "</div>\n");
} ]);

"use strict";

var MetadataReader = function() {
    MetadataReader.$inject = [ "$http", "$q", "metaDataBaseUrl" ];
    function MetadataReader($http, $q, metaDataBaseUrl) {
        this.$http = $http;
        this.$q = $q;
        this.metaDataBaseUrl = metaDataBaseUrl;
        this.resourcesUrl = metaDataBaseUrl + "resources/";
    }
    MetadataReader.prototype.readRaw = function(table, key) {
        return this.$http.get("" + this.resourcesUrl + table + "/" + encodeURIComponent(key) + ".raw.js", {
            cache: true
        });
    };
    MetadataReader.prototype.readZip = function(name) {
        return this.readZipAt(this.resourcesUrl + name + ".zip.js");
    };
    MetadataReader.prototype.readZipAt = function(path) {
        var defer = this.$q.defer();
        JSZipUtils.getBinaryContent(path, function(err, data) {
            if (err) {
                defer.reject(err);
            } else {
                defer.resolve(new JSZip(data));
            }
        });
        return defer.promise;
    };
    return MetadataReader;
}();

angular.module("siteGeneratorStaticsMetadataApp").service("metadataReader", MetadataReader);