# my-work

import $ from 'jquery';
import { queryParams } from '../../scripts';
import sanitizeHtml from 'sanitize-html';

let currentPage = 0;
const resultCountPerPage = 10;

var searchKeyword = queryParams.decode('searchKeyword');
const campusLocation = queryParams.decode('campusLocation');
const courseType = queryParams.decode('courseType');
const studentType = queryParams.decode('studentType');
const isGlobalSearch = queryParams.decode('isGlobalSearch');
const searchResultUrl = queryParams.decode('searchResultUrl');
const debugErrorEnabled = false;
const genericPages = "genericPages";
const channelName = $("body").attr("data-channelname");

const searchConst = {
    CAMPUS: 'All campuses', 
    COURSE: 'All course types',
    INTERNATIONAL: 'International Student',
    ALL: 'all',
    COURSES: 'courses',
    NEWS: 'news',
    BLOGS: 'blogs',

    OBJ_ALL: 'All',
    OBJ_CONTENT: 'Content',
    OBJ_COURSES: 'Courses',
    OBJ_NEWS: 'News'
};

const viewTemplate = {
    COURSE: [
            '/conf/bendigokangan/settings/wcm/templates/course-page',
            '/conf/bendigokangan/settings/wcm/templates/kangan-course-page',
            '/conf/bendigokangan/settings/wcm/templates/bendigo-course-page'
        ],
    PAGECONTENT: '/conf/bendigokangan/settings/wcm/templates/page-content',
    NEWS: '/conf/bendigokangan/settings/wcm/templates/news-page',
};

let currentType = searchConst.OBJ_ALL;

const content = {
    
    SEARCHING: '<strong>Searching...</strong>',
    SOMETHINGWRONG: '<strong>Something went wrong!</strong>',
    SEARCHDATA: (searchData, start, end) => {
        var searchStr = '';
        if(searchData.length) {
            searchStr = searchStr + ` <strong>${searchData.length}</strong> results found`;
        } else {
            searchStr = searchStr + ` <strong>No results found</strong>`
        }
        if(start && end) {
            searchStr = `<strong>${start}-${end}</strong> of ` + searchStr;
        }
        if(searchKeyword) {
            searchStr = searchStr + ` for <strong>"${searchKeyword}"</strong>`;
        } 
        if(((campusLocation || courseType) && ((campusLocation !== searchConst.CAMPUS) || (courseType !== searchConst.COURSE)))) {
            searchStr = searchStr + (searchKeyword ? ` and` : ` for`);
        }
        if(campusLocation && (campusLocation !== searchConst.CAMPUS)) {
            searchStr = searchStr + ` Campus Location: <strong>"${campusLocation}"</strong>`;
            if(courseType && (courseType !== searchConst.COURSE)) {
                searchStr = searchStr + ` and`;
            }
        }
        if(courseType && (courseType !== searchConst.COURSE)) {
            searchStr = searchStr + ` Course Type: <strong>"${courseType}"</strong>`;
        }
        return searchStr;
    }

    
};



export const searchCtrl = async ($ele) => {
    debugger;
    const $searchResultBar = $('.result-search-bar, .find-course-search');
    const searchPageUrl = searchResultUrl.replace(/(.html)/g, '');

    // console.log(searchKeyword);
    // searchKeyword = await sanitizeHtml(searchKeyword,{
    //     allowedTags: [], // Strip all tags
    //     allowedAttributes: {} // No attributes allowed
    //   });
    // console.log(searchKeyword);

    const payloadObject = {
        q: searchKeyword,
        searchPageUrl: searchResultUrl.replace(/(.html)/g, '') || ''
    };




    function sanitizeInput(input) {
    // searchKeyword = encodeURI(searchKeyword);
    const parser = new DOMParser();
    const doc = parser.parseFromString(searchKeyword, 'text/html');
    return doc.body.textContent || "";
}
  
   searchKeyword = sanitizeInput(encodeURI(searchKeyword));

    if(studentType && (studentType === searchConst.INTERNATIONAL)) {
        payloadObject.isInternational = true;
    }
    if(campusLocation && (campusLocation !== searchConst.CAMPUS)) {
        payloadObject.campusLocation = campusLocation;
    }
    if(courseType && (courseType !== searchConst.COURSE)) {
        payloadObject.courseType = courseType;
    }
    if(isGlobalSearch) {
        payloadObject.isGlobalSearch = isGlobalSearch;
        $searchResultBar.closest('.container.responsivegrid').remove();
    }
    
    paginationHandler($ele, []);
    showResultMessage($ele, content.SEARCHING);
    let uri = `/bin/searchResults?${queryParams.encode(payloadObject)}`;

    try {
        const response = await fetch(uri);
        const resJson = await response.json();
        let resData = isGlobalSearch ? resJson : { [searchConst.OBJ_COURSES]: resJson };

        for(const key in resData) {
            resData[key] = convertDataParsing(key, resData[key]);
        }

        if(isGlobalSearch) {
            resData[searchConst.OBJ_ALL] = [
                ...(resJson[searchConst.OBJ_CONTENT] || []),
                ...(resJson[searchConst.OBJ_COURSES] || []),
                ...(resJson[searchConst.OBJ_NEWS] || []),
            ].sort((a, b) => (a['pageTitle'] || a['jcr:title']) > (b['pageTitle'] || b['jcr:title']) ? 1 : -1 );
        }

        const analyticPayload = {
            "event": "pageLoaded",
            "channel": channelName + " | search",
            "searchTerm": searchKeyword,
            "searchResults": resData[isGlobalSearch ? searchConst.OBJ_ALL : searchConst.OBJ_COURSES].length,
            "searchFilter": campusLocation + ' | ' + courseType,
            "parentId": Object.keys(window.adobeDataLayer?.getState()?.page || {})[0]
        };
        window.adobeDataLayer?.push({"page": {
            [$('body').attr('id') || genericPages] : analyticPayload,
        }});

        debugErrorEnabled && console.log(analyticPayload);
        debugErrorEnabled && console.log(resData);
        
        handleTab($ele, {...resData});
    } catch(e) {
        debugErrorEnabled && console.log(e);
        showResultMessage($ele, content.SOMETHINGWRONG);
    }
};

const convertDataParsing = (key, resData) => {
    if(!resData) {
        return false;
    }
    
    if(key !== searchConst.OBJ_COURSES) {
        return JSON.parse(resData).map((res, index) => {
            if(!res) {
                return false;
            }
            const btnid = "moreinfo-" + key+index;
            let pagePath = res['pagePath'];
            if((res['pagePath']||'').includes('.html')) {
                pagePath = res?.pagePath?.replace(/^(\/content\/)(.*)(\/en)/g, '')
            }
            res['pagePath'] = pagePath;
            res.btnid = btnid;
            return res;
        });
    }

    return (isGlobalSearch ? JSON.parse(resData) : resData)
        .filter((res) => {
            try {
                const parsedJSON = $.parseJSON(res);
                return parsedJSON['enrolmentType'];
            } catch(e) {
                debugErrorEnabled && console.log(e)
                return false;
            }
        }).map((res, index) => {
            try {
                const parsedJSON = $.parseJSON(res);
                let pagePath = parsedJSON?.pagePath;
                if((parsedJSON?.pagePath||'').includes('.html')) {
                    pagePath = parsedJSON?.pagePath?.replace(/^(\/content\/)(.*)(\/en\/)/g, '/')
                }
                parsedJSON.pagePath = pagePath;
                parsedJSON.contents = $.parseJSON(parsedJSON.contents || 'false');
                parsedJSON.courseFees = $.parseJSON(parsedJSON.courseFees || 'false') || [];
                parsedJSON.units = $.parseJSON(parsedJSON.units || 'false') || [];
                parsedJSON.upcomingCourses = $.parseJSON(parsedJSON.upcomingCourses || 'false') || [];
                parsedJSON.indexId = index;
                parsedJSON.btnid = "moreinfo-" + key+index;
                if(parsedJSON.upcomingCourses && parsedJSON.upcomingCourses.length) {
                    const location = (parsedJSON.upcomingCourses||[])
                        .filter(v => !!v.location && (v.enabled === "true"))
                        .map(v => v.location);
                    const locationSet = Array.from(new Set(location) || []).filter(v => !!v);                  
                    parsedJSON.upcomingCoursesLocation = Array.from(locationSet).join(', ');
                }
                const btnid = "moreinfo-" + key+index;
                parsedJSON.btnid = btnid;

                return parsedJSON;
            } catch (e) {
                debugErrorEnabled && console.log(e);
                return false;
            }
        })
        .filter(v => v);
}

const scrollToTop = ($ele) => {
    setTimeout(() => {
        window.scroll({
            top: $ele.find('.search-results-wrapper').offset().top,
            behavior: 'smooth'
        });
    });
}

const showResultMessage = ($ele, msg) => {
    const $showCountForResult = $ele.find('#showCountForResult');
    $showCountForResult.html(msg);
};

const handleTab = ($ele, searchData) => {
    const $tabLink = $ele.find('.search-results-tab-link');
    const $tabContainer = $ele.find('#searchResultHeadTab');
    const $departmentTile = $ele.find('.department-list');
    if(isGlobalSearch) {
        $tabContainer.removeClass('d-none');
    }

    if(isGlobalSearch) {
        $departmentTile.removeClass('col-lg-9');
    }

    paginationHandler($ele, searchData[searchConst[isGlobalSearch ? 'OBJ_ALL' : 'OBJ_COURSES']]);
    handleFacetFilter($ele, searchData[searchConst[isGlobalSearch ? 'OBJ_ALL' : 'OBJ_COURSES']]);
    renderSearchItems($ele, searchData[searchConst[isGlobalSearch ? 'OBJ_ALL' : 'OBJ_COURSES']].slice(currentPage * resultCountPerPage, resultCountPerPage + currentPage * resultCountPerPage));

    $tabLink.on('click', (e) => {
        const $currentElement = $(e.currentTarget);
        let dataToFilter = [];
        currentType = $currentElement.attr('data-value');
        currentPage = 0;
        switch(currentType) {
            case searchConst.OBJ_NEWS:
                dataToFilter = searchData[currentType].filter((v) => {
                    return viewTemplate.COURSE.indexOf(v[isGlobalSearch ? 'pageType' : 'cq:template']) === -1;
                });
                $departmentTile.removeClass('col-lg-9');
                break;
            case searchConst.OBJ_COURSES:
                dataToFilter = searchData[currentType].filter((v, i) => {
                    return viewTemplate.COURSE.indexOf(v['cq:template']) !== -1;
                });
                $departmentTile.addClass('col-lg-9');
                break;
            default:
                $departmentTile.removeClass('col-lg-9');
                dataToFilter = searchData[currentType];
                break;
        }
        $tabLink.removeClass('active');
        $currentElement.addClass('active');

        paginationHandler($ele, dataToFilter);
        handleFacetFilter($ele, dataToFilter);
        renderSearchItems($ele, dataToFilter.slice(currentPage * resultCountPerPage, resultCountPerPage + currentPage * resultCountPerPage));
    });
};

const handleFacetFilter = ($ele, searchData) => {
    const isCourseSelected = currentType === searchConst.OBJ_COURSES;
    const $facetTemplate = $ele.find('#searchFacetTemplate');
    const $facetItemTemplate = $ele.find('#searchFacetItemTemplate');
    const $facetRenderBox = $ele.find('#searchFacetRenderContainer');
    const $clearBtn = $ele.find('#clearFilterResults');

    if(!searchData.length || (isGlobalSearch && !isCourseSelected)) {
        $facetRenderBox.html('');
        $facetRenderBox.parent().hide();
        $clearBtn.hide();

        return false;
    }
    
    $facetRenderBox.parent().show();

    const filterSections = {
        Course: {
            title: "Enrolment Type", 
            filters: new Set(),
            selected: new Set()
        }
    };

    let facetTemplate = $facetTemplate.text();
    let facetItemTemplate = $facetItemTemplate.text();

    const mappingItems = {
        FACETTITLE: /{{ntt-facetTitle}}/g,
        ID: /{{ntt-itemid}}/g,
        NAME: /{{ntt-itemname}}/g,
        TITLE: /{{ntt-itemtitle}}/g,
        ITEMRENDERBOX: /{{ntt-searchFacetItemRenderBox}}/g,
        VALUE: /{{ntt-itemvalue}}/g
    };

    let facetToRender = [];

    Object.keys(filterSections).forEach((key, index) => {
        const val = filterSections[key];
        let template = facetTemplate;
        for (const pindex in searchData) {
            const currentData = searchData[pindex];
            const {
                contents,
                courseFees,
                units,
                upcomingCourses
            } = currentData;
            template = template.replace(mappingItems.FACETTITLE, val.title);
            switch(key) {
                case 'Course': 
                    if(currentData['enrolmentType']) {
                        val.filters.add(currentData['enrolmentType']);
                    }
                    break;
            }
        }

        let itemTemplate = Array.from(val.filters).map((item) => {
            var tempStr = facetItemTemplate;
            tempStr = tempStr
                .replace(mappingItems.ID, `ctrl-list_${index}-${item}`)
                .replace(mappingItems.NAME, `ctrl-list_index-${index}`)
                .replace(mappingItems.VALUE, `${key}-[.]-${item}`)
                .replace(mappingItems.TITLE, item);
            return tempStr;
        });

        template = template.replace(mappingItems.ITEMRENDERBOX, `<ul class="facet-checkbox-list" id="item-checkbox-list-${index}">${itemTemplate.join('')}</ul>`);
        template = `<div class="facet-title-container" id="facet-search-block-${index}">${template}</div>`
        facetToRender.push(template);
    });

    $facetRenderBox.html(facetToRender.join(''));

    setTimeout(() => {
        const $facetCheckbox = $ele.find('.facet-checkbox-input');

        $facetCheckbox.on('change', (event) => {
            const {value, checked} = event.target;
            const [key, item] = value.split('-[.]-');
            let filteredResponse = [...searchData];
            let allElementsAreChecked = true;

            if(checked) {
                filterSections[key].selected.add(item); 
                $clearBtn.show();
            } else {
                filterSections[key].selected.delete(item);
                $facetCheckbox.each((index, element) => {
                    if(element.checked) {
                        allElementsAreChecked = false;
                    }
                });
                allElementsAreChecked ? $clearBtn.hide() : $clearBtn.show();
            }

            let haveAnyValueSelected = false;

            Object.values(filterSections).forEach((innerval) => {
                haveAnyValueSelected = haveAnyValueSelected || !!innerval.selected.size;
            });
            
            if(haveAnyValueSelected) {
                filteredResponse = [...searchData].filter(val => {
                    const filterDataList = Array.from(filterSections[key].selected);
                    let isValid = false;
                    Object.keys(filterSections).forEach((innerval) => {
                        let innerkey = '';
                        switch(innerval) {
                            case 'Course': 
                                innerkey = 'enrolmentType'
                                break;
                            case 'Qualification':
                                innerkey = 'qualificationLevel'
                                break;
                        }
                        isValid = isValid || filterDataList.some(someval => {
                            return innerval === 'Campuses' ? (val?.[innerkey]||'').includes(someval) : (val?.[innerkey] === someval);
                        });
                    });
                    return isValid;
                });
            }

            currentPage = 0;
            renderSearchItems($ele, [...filteredResponse].slice(currentPage * resultCountPerPage, resultCountPerPage + currentPage * resultCountPerPage));
            paginationHandler($ele, [...filteredResponse]);
        });

        $clearBtn.on('click', (e) => {
            e.preventDefault();
            $facetCheckbox.each((index, element) => {
                if(element.checked) {
                    $(element).trigger('click');
                }
            });
            $clearBtn.hide();
        });
    }, 50);

    setTimeout(() => {
        if(studentType === searchConst.INTERNATIONAL) {
            $ele.find('.facet-checkbox-input[value="Course-[.]-International"]').trigger('click');
        }
    }, 100);

    return searchData;
};

const paginationHandler = ($ele, searchData) => {
    let maxCount = Math.floor(searchData.length / resultCountPerPage);
    const $prevButton = $ele.find('#searchPrevButton');
    const $nextButton = $ele.find('#searchNextButton');
    if(window.location.search && searchData && searchData.length) {
        showResultMessage($ele, content.SEARCHDATA(searchData, currentPage * resultCountPerPage + 1, ((currentPage+1) * resultCountPerPage) >= searchData.length ? searchData.length : (currentPage+1) * resultCountPerPage));
    } else {
        showResultMessage($ele, content.SEARCHDATA([]));
    }
    if(!searchData.length) {
        $prevButton.hide();
        $nextButton.hide();
        return false;
    }

    $prevButton.show();
    $nextButton.show();

    $prevButton.attr('disabled', 'true');
    if(searchData.length <= resultCountPerPage) {
        $nextButton.attr('disabled', 'true');
    } else {
        $nextButton.removeAttr('disabled', 'true');
    }

    $prevButton
        .off('click')    
        .on('click', (e) => {
            e.preventDefault();
            --currentPage;
            if (currentPage < 0) {
                ++currentPage;
                return false;
            }
            if (currentPage === 0) {
                $prevButton.attr('disabled', 'true');
            }
            $nextButton.removeAttr('disabled');
            renderSearchItems($ele, searchData.slice(currentPage * resultCountPerPage, (currentPage+1) * resultCountPerPage));
            showResultMessage($ele, content.SEARCHDATA(searchData, currentPage * resultCountPerPage + 1, ((currentPage+1) * resultCountPerPage) >= searchData.length ? searchData.length : (currentPage+1) * resultCountPerPage));
            scrollToTop($ele);
        });

    $nextButton
        .off('click')    
        .on('click', (e) => {
            e.preventDefault();
            ++currentPage;
            if (currentPage > maxCount) {
                --currentPage;
                return false;
            }
            if (currentPage === maxCount) {
                $nextButton.attr('disabled', 'true');
            }
            $prevButton.removeAttr('disabled');
            renderSearchItems($ele, searchData.slice(currentPage * resultCountPerPage, (currentPage+1) * resultCountPerPage));            
            showResultMessage($ele, content.SEARCHDATA(searchData, currentPage * resultCountPerPage + 1, ((currentPage+1) * resultCountPerPage) >= searchData.length ? searchData.length : (currentPage+1) * resultCountPerPage));
            scrollToTop($ele);
        });
    return searchData;
};

const renderSearchItems = ($ele, searchData) => {
    const $resultRenderBox = $ele.find('#searchResultRenderContainer');
    const $resultTemplate = $ele.find('#searchResultTemplate');
    const $internationalTemplate = $ele.find('#internationalTemplate');
    const $resultNewsTemplate = $ele.find('#searchResultBlogsTemplate');
    if(!searchData.length) {
        $resultRenderBox.html('');
    }
    const resultTemplate = $resultTemplate.text();
    const resultNewsTemplate = $resultNewsTemplate.text();
    const internationalTemplate = $internationalTemplate.text();

    const isUserOnPublished = $ele.find('#userMode')?.attr('data-mode') === 'published';
    const mappingItems = {
        PATH: /{{ntt-path}}/g,
        HEADING: /{{ntt-heading}}/g,
        DESCRIPTION: /{{ntt-description}}/g,
        CODE: /{{ntt-code}}/g,
        LENGTH: /{{ntt-length}}/g,
        LOCATION: /{{ntt-location}}/g,
        APPLYHERE: /{{ntt-applyHere}}/g,
        CONTACT: /{{ntt-contact}}/g,
        CRICOS: /{{ntt-cricos}}/g,
        SHOWCRICOS: /{{ntt-showcricos}}/g,
        DATALAYER: /{{ntt-datalayer}}/g,
        BTNID: /{{ntt-btnid}}/g,
        HAVEINTERNATIONAL: /{{ntt-haveInternational}}/g,
    };
    let dataToRender = [];

    for (const pindex in searchData) {
        const currentData = searchData[pindex];

        const {
            contents,
            courseFees,
            units,
            btnid,
            upcomingCourses,
            enrolmentType,
            upcomingCoursesLocation
        } = currentData;

        let pagePath = currentData['pagePath'];
        const durationLength = [];

        if(contents && contents['durationFullTime']) {
            durationLength.push(contents['durationFullTime']);
        }
        if(contents && contents['durationOnline']) {
            durationLength.push(contents['durationOnline']);
        }
        if(contents && contents['durationFlex']) {
            durationLength.push(contents['durationFlex']);
        }
        if(contents && contents['durationPartTime']) {
            durationLength.push(contents['durationPartTime']);
        }

        if(isUserOnPublished) {
            pagePath = currentData?.pagePath?.replace(/^(\/content\/)(.*)(\/en\/)/g, '/')
        }

        let currentTemplate =  viewTemplate.COURSE.indexOf(currentData['cq:template']) !== -1 ? resultTemplate : resultNewsTemplate;
        
        currentTemplate = currentTemplate
            .replace(mappingItems.PATH, pagePath || '-')
            .replace(mappingItems.HEADING, currentData['pageTitle'] || currentData['jcr:title'] || '-')
            .replace(mappingItems.DESCRIPTION, 
                contents?.courseDescription || 
                contents?.courseOverView || 
                currentData['pageDescription'] ||
                currentData['jcr:description'] ||
            '-')
            .replace(mappingItems.CODE, currentData['courseCode'] || '-')
            .replace(mappingItems.LENGTH, durationLength.length ? durationLength.join(', '): '-')
            .replace(mappingItems.APPLYHERE, pagePath || '-')
            .replace(mappingItems.CONTACT, currentData[''] || '-')
            .replace(mappingItems.SHOWCRICOS, (currentData['isInternational'] && currentData['cricosCode']) || 'd-none')
            .replace(mappingItems.CRICOS, currentData['cricosCode'] || '-')
            .replace(mappingItems.LOCATION, upcomingCoursesLocation || '-')
            .replace(mappingItems.BTNID, btnid || '-')
            .replace(mappingItems.HAVEINTERNATIONAL, enrolmentType === 'International' ? internationalTemplate : '');

        dataToRender.push(currentTemplate);

        window.adobeDataLayer?.push({"component": {
            [btnid]: {
                "@type": "bendigokangan/components/button",
                "dc:title": currentData['pageTitle'] || currentData['jcr:title'] + "- More Info",
                "repo:modifyDate": currentData['cq:lastModified'],
                "xdc:linkURL": pagePath,
                "parentId": Object.keys(window.adobeDataLayer?.getState()?.page || {})[0]
            }
        }});
    }
    $resultRenderBox.html(dataToRender.join(''));

    setTimeout(() => {
        $ele.find('.search-more-info-btn').off('click').on('click', function (e) {
            e.preventDefault();
            window.adobeDataLayer.push({
                "event": "cmp:customClick",
                "eventInfo": {
                    "path": $(e.currentTarget).attr('id')
                }
            });
            window.location.href = $(e.currentTarget).attr("href");
        });
    }, 50);
};
