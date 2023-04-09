---
title: Recall Tracing Token
description: A standardized token for managing recalls in a supply chain that enables efficient traceability and communication between manufacturers, distributors, and customers.
author: Christian Ziegler (@Totenfluch), Norman Pytell
discussions-to: TODO
status: Draft
type: Standards Track
category: ERC
created: 2023-03-21
requires: 1155, 163
---

## Abstract

This token standard enables tracking of products in a supply chain, facilitating traceability and communication of defects and recalls. It allows users to announce defects, check tokens, and forward recalls based on unique product IDs (TokenIDs). This standard supports incentive structures for faster recall responses and provides efficient recall management, communication, and traceability between various stakeholders in the supply chain, including customers.

## Motivation

Supply chain management often faces challenges in tracking and managing product recalls efficiently. Existing solutions can be cumbersome, slow, and costly. The token standard aims to improve the recall process, communication, and traceability between different parties in the supply chain. This standard also helps with managing customer integration and incentivizes faster responses to recalls.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

Please see the provided Solidity interface for the detailed function specifications, events, and data structures.

```
 /** @dev 
        The `_announcer` argument MUST be the address of an account/contract that currently owns the token
        The `_tokenId` argument MUST be the token being announced defect
    */
    event DefectAnnounced(address indexed _announcer, uint256 _tokenId);

    /** @dev 
        The `_announcer` argument MUST be the address of an account/contract approved to manage the tokens
        The `_tokenIds` argument MUST be the tokens being announced defect    
    */
    event ForwardRecall(address indexed _announcer, uint256[] _tokenIds);

    /** @dev 
        The `_announcer` argument MUST be the address of an account/contract approved to manage the token
        The `_tokenId` argument MUST be the token being announced defect
        The `_resultState` argument must be {CHECKED_NO_DEFECT, CHECKED_DEFECT}
    */
    event TokenChecked(
        address indexed _announcer,
        uint256 _tokenId,
        TokenCheckingState _resultState
    );

    /** @dev 
        The `_announcer` argument MUST be the address of an account/contract approved to manage the token
        The `_tokenId` argument MUST be the token that the manufacturers are merged into
        The `_mergedTokenId` MUST be the token that the manufacturers are sourced from
    */
    event TokenMerged(
        address indexed _announcer,
        uint256 _tokenId,
        uint256 _mergedTokenId
    );

    enum TokenState {
        OK,
        ON_HOLD,
        NOT_OK
    }

    enum TokenCheckingState {
        NONE,
        PLEASE_CHECK,
        CHECKED_NO_DEFECT,
        CHECKED_DEFECT
    }

    /**
        @notice Changes the `TokenState` for a token specified by `_tokenId` to `{NOT_OK}`
        @dev Caller must be approved to manage the token
        MUST revert if TokenState of `_tokenId` is {ON_HOLD, NOT_OK}.
        MUST revert on any other error.
        MUST emit the `DefectAnnounced` event to reflect the TokenState change     
        @param _tokenId   The defect Token
    */
    function announceDefect(uint256 _tokenId) external;

    /**
        @notice Changes the `TokenCheckingState` for a token specified by `_tokenId` to `_tokenCheckingState`
        @dev Caller must be approved to manage the token
        MUST revert if `TokenCheckingState` of `_tokenId` is `{NONE, CHECKED_NO_DEFECT, CHECKED_DEFECT}`.
        MUST revert on any other error.
        MUST emit the `TokenChecked` event to reflect the TokenCheckingState change     
        @param _tokenId             The defect Token
        @param _tokenCheckingState  Result state of the check
    */
    function checkToken(
        uint256 _tokenId,
        TokenCheckingState _tokenCheckingState
    ) external;

    /**
        @notice Changes the `TokenState` for all token specified by `_tokenIds` to `{NOT_OK}`
        @dev Caller must be approved to manage the tokens
        MUST revert on any other error.
        MUST emit the `ForwardRecall` event to reflect the TokenState change     
        @param _tokenIds           The defect Token
    */
    function forwardRecall(uint256[] calldata _tokenIds) external;

    /** @dev 
        The `_tokenIdMergeTo` argument MUST be the token that the manufacturers are merged into
        The `_tokenIdMergeSource` MUST be the token that the manufacturers are sourced from
    */
    function mergeToken(uint256 _tokenIdMergeTo, uint256 _tokenIdMergeSource) external;

    /** @dev 
        The `_tokenContractAddress` MUST be a ERCxxx contract
        The `_tokenIdMergeTo` argument MUST be the token that the manufacturers are merged into
        The `_tokenIdMergeSource` MUST be the token that the manufacturers are sourced from
    */
    function mergeExternalToken(address _tokenContractAddress, uint256 _tokenIdMergeTo, uint256 _tokenIdMergeSource) external;
```

## Backwards Compatibility

No backward compatibility issues found.

## Reference Implementation

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.18;

import "@openzeppelin/contracts/token/ERC1155/ERC1155.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

/**
    @title ERC-ERCxxxx Recall Tracing Token
    @dev See https://eips.ethereum.org/EIPS/eip-ERCxxxx
    @notice This contract represents a recall tracing token standard that allows manufacturers, distributors, and customers
            to announce defects, check tokens, and forward recalls based on unique product IDs (TokenIDs). 
            It enables efficient traceability and communication within a product supply chain.
 */

contract RecallToken is ERC1155, Ownable {
    /** @dev 
        The `_announcer` argument MUST be the address of an account/contract that currently owns the token
        The `_tokenId` argument MUST be the token being announced defect
    */
    event DefectAnnounced(address indexed _announcer, uint256 _tokenId);

    /** @dev 
        The `_announcer` argument MUST be the address of an account/contract approved to manage the tokens
        The `_tokenIds` argument MUST be the tokens being announced defect    
    */
    event ForwardRecall(address indexed _announcer, uint256[] _tokenIds);

    /** @dev 
        The `_announcer` argument MUST be the address of an account/contract approved to manage the token
        The `_tokenId` argument MUST be the token being announced defect
        The `_resultState` argument must be {CHECKED_NO_DEFECT, CHECKED_DEFECT}
    */
    event TokenChecked(
        address indexed _announcer,
        uint256 _tokenId,
        TokenCheckingState _resultState
    );

    /** @dev 
        The `_announcer` argument MUST be the address of an account/contract approved to manage the token
        The `_tokenId` argument MUST be the token that the manufacturers are merged into
        The `_mergedTokenId` MUST be the token that the manufacturers are sourced from
        The `_manufacturers` MUST all the distinct array combination of manufacturers of _tokenId and _mergedTokenId
    */
    event TokenMerged(
        address indexed _announcer,
        uint256 _tokenId,
        uint256 _mergedTokenId,
        address[] _manufacturers
    );

    enum TokenState {
        OK,
        ON_HOLD,
        NOT_OK
    }

    enum TokenCheckingState {
        NONE,
        PLEASE_CHECK,
        CHECKED_NO_DEFECT,
        CHECKED_DEFECT
    }

    mapping(uint256 => address[]) manufacturers;

    function getManufacturersOfToken(
        uint256 _tokenId
    ) public view returns (address[] memory) {
        return manufacturers[_tokenId];
    }

    mapping(address => mapping(uint256 => TokenCheckingState)) manufacturerTokenCheckingStates;

    function getManufacturerTokenCheckingStateValue(
        address _address,
        uint256 _tokenId
    ) public view returns (TokenCheckingState) {
        return manufacturerTokenCheckingStates[_address][_tokenId];
    }

    mapping(uint256 => TokenCheckingState) public tokenCheckingStates;

    function getTokenCheckingState(
        uint256 _tokenId
    ) public view returns (TokenCheckingState) {
        return tokenCheckingStates[_tokenId];
    }

    mapping(uint256 => TokenState) public tokenStates;

    function getTokenStateValue(
        uint256 _tokenId
    ) public view returns (TokenState) {
        return tokenStates[_tokenId];
    }

    mapping(uint256 => bool) public inProduction;

    function getInProductionValue(uint256 _tokenId) public view returns (bool) {
        return inProduction[_tokenId];
    }

    modifier _isManufacturer(uint256 _tokenId) {
        bool tempIsManufacturer = false;
        for (uint i = 0; i < manufacturers[_tokenId].length; i++) {
            if (_msgSender() == manufacturers[_tokenId][i]) {
                tempIsManufacturer = true;
                break;
            }
        }

        require(tempIsManufacturer == true, "Must be Manufacturer");
        _;
    }

    function _authorizedForToken(address _from, uint256 _tokenId) public view {
        require(balanceOf(_msgSender(), _tokenId) >= 1, "No Token owned");
        require(
            _from == _msgSender() || isApprovedForAll(_from, _msgSender()),
            "Caller is not Token Owner or approved"
        );
    }

    constructor(string memory uri) ERC1155(uri) {}

    /**
        @notice Changes the `TokenState` for a token specified by `_tokenId` to `{NOT_OK}`
        @dev Caller must be approved to manage the token
        MUST revert if TokenState of `_tokenId` is {ON_HOLD, NOT_OK}.
        MUST revert on any other error.
        MUST emit the `DefectAnnounced` event to reflect the TokenState change     
        @param _tokenId   The defect Token
    */
    function announceDefect(uint256 _tokenId) public {
        _authorizedForToken(_msgSender(), _tokenId);
        bool isManufacturer = false;
        uint selectedManufacturer = 0;
        for (uint i = 0; i < manufacturers[_tokenId].length; i++) {
            if (_msgSender() == manufacturers[_tokenId][i]) {
                isManufacturer = true;
                selectedManufacturer = i;
                break;
            }
        }
        if (isManufacturer) {
            tokenStates[_tokenId] = TokenState.NOT_OK;
            emit DefectAnnounced(_msgSender(), _tokenId);
            return;
        } else {
            tokenStates[_tokenId] = TokenState.ON_HOLD;
        }

        for (uint i = 0; i < manufacturers[_tokenId].length; i++) {
            manufacturerTokenCheckingStates[manufacturers[_tokenId][i]][
                _tokenId
            ] = TokenCheckingState.PLEASE_CHECK;
        }
        emit DefectAnnounced(_msgSender(), _tokenId);
    }

    /**
        @notice Changes the `TokenCheckingState` for a token specified by `_tokenId` to `_tokenCheckingState`
        @dev Caller must be approved to manage the token
        MUST revert if `TokenCheckingState` of `_tokenId` is `{NONE, CHECKED_NO_DEFECT, CHECKED_DEFECT}`.
        MUST revert on any other error.
        MUST emit the `TokenChecked` event to reflect the TokenCheckingState change     
        @param _tokenId             The defect Token
        @param _tokenCheckingState  Result state of the check
    */
    function checkToken(
        uint256 _tokenId,
        TokenCheckingState _tokenCheckingState
    ) public _isManufacturer(_tokenId) {
        uint selectedManufacturer = 0;
        for (uint i = 0; i < manufacturers[_tokenId].length; i++) {
            if (_msgSender() == manufacturers[_tokenId][i]) {
                selectedManufacturer = i;
                break;
            }
        }
        require(
            manufacturerTokenCheckingStates[_msgSender()][_tokenId] ==
                TokenCheckingState.PLEASE_CHECK,
            "Token can not be checked"
        );
        manufacturerTokenCheckingStates[_msgSender()][
            _tokenId
        ] = _tokenCheckingState;
    }

    /**
        @notice Changes the `TokenState` for all token specified by `_tokenIds` to `{NOT_OK}`
        @dev Caller must be approved to manage the tokens
        MUST revert on any other error.
        MUST emit the `ForwardRecall` event to reflect the TokenState change     
        @param _tokenIds           The defect Token
    */
    function forwardRecall(uint256[] calldata _tokenIds) public {
        for (uint i = 0; i < _tokenIds.length; i++) {
            bool tempIsManufacturer = false;
            for (uint x = 0; x < manufacturers[_tokenIds[i]].length; x++) {
                if (_msgSender() == manufacturers[_tokenIds[i]][x]) {
                    tempIsManufacturer = true;
                    break;
                }
            }
            require(
                tempIsManufacturer == true,
                "Not a Manufacturer of this Token"
            );
        }
        for (uint i = 0; i < _tokenIds.length; i++) {
            tokenStates[_tokenIds[i]] = TokenState.NOT_OK;
        }
        emit ForwardRecall(_msgSender(), _tokenIds);
    }

    /**
     * @dev See {IERC1155-safeTransferFrom}.
     * if _internal is true, receiver will be added as manufacturer, otherwise not
     * after _internal is set false for one transfer, receivers will no longer be added as manufacturers
     */
    function transferRecallToken(
        address _receiver,
        uint256 _tokenId,
        uint256 amount,
        bytes memory data,
        bool _internal
    ) public {
        safeTransferFrom(_msgSender(), _receiver, _tokenId, amount, data);
        if (inProduction[_tokenId] && _internal) {
            appendUniqueManufacturerOfToken(_receiver, _tokenId);
        }
        if (!_internal && !inProduction[_tokenId]) {
            inProduction[_tokenId] = false;
        }
    }

    /**
     * @dev See {IERC1155-safeBatchTransferFrom}.
     * if _internal is true, receiver will be added as manufacturer, otherwise not
     * after _internal is set false for one transfer, receivers will no longer be added as manufacturers
     */
    function batchTransferRecallToken(
        address _receiver,
        uint256[] calldata _tokenIds,
        uint256[] calldata _amounts,
        bytes memory data,
        bool _internal
    ) public {
        safeBatchTransferFrom(
            _msgSender(),
            _receiver,
            _tokenIds,
            _amounts,
            data
        );
        for (uint256 i = 0; i < _tokenIds.length; i++) {
            if (inProduction[_tokenIds[i]] && _internal) {
                appendUniqueManufacturerOfToken(_receiver, _tokenIds[i]);
            }
            if (!_internal && !inProduction[_tokenIds[i]]) {
                inProduction[_tokenIds[i]] = false;
            }
        }
    }

    /**
     * @dev Creates `amount` tokens of token type `id`, and assigns them to `to`.
     *
     * Emits a {TransferSingle} event.
     *
     * Requirements:
     *
     * - `to` cannot be the zero address.
     * - If `to` refers to a smart contract, it must implement {IERC1155Receiver-onERC1155Received} and return the
     * acceptance magic value.
     */
    function mint(
        address to,
        uint256 id,
        uint256 amount,
        bytes memory data
    ) public onlyOwner {
        _mint(to, id, amount, data);
        inProduction[id] = true;
        manufacturers[id].push(_msgSender());
    }

    /**
     * @dev xref:ROOT:erc1155.adoc#batch-operations[Batched] version of {_mint}.
     *
     * Emits a {TransferBatch} event.
     *
     * Requirements:
     *
     * - `ids` and `amounts` must have the same length.
     * - If `to` refers to a smart contract, it must implement {IERC1155Receiver-onERC1155BatchReceived} and return the
     * acceptance magic value.
     */
    function mintBatch(
        address to,
        uint256[] memory ids,
        uint256[] memory amounts,
        bytes memory data
    ) public onlyOwner {
        _mintBatch(to, ids, amounts, data);
        for (uint256 i = 0; i < ids.length; i++) {
            inProduction[ids[i]] = true;
            manufacturers[ids[i]].push(_msgSender());
        }
    }

    /** @dev 
        The `_tokenIdMergeTo` argument MUST be the token that the manufacturers are merged into
        The `_tokenIdMergeSource` MUST be the token that the manufacturers are sourced from
    */
    function mergeToken(
        uint256 _tokenIdMergeTo,
        uint256 _tokenIdMergeSource
    ) public _isManufacturer(_tokenIdMergeTo) {
        _authorizedForToken(_msgSender(), _tokenIdMergeTo);
        //_mergeInto does not need to be owned
        for (
            uint256 i = 0;
            i < manufacturers[_tokenIdMergeSource].length;
            i++
        ) {
            appendUniqueManufacturerOfToken(
                manufacturers[_tokenIdMergeSource][i],
                _tokenIdMergeTo
            );
        }
        emit TokenMerged(
            _msgSender(),
            _tokenIdMergeTo,
            _tokenIdMergeSource,
            manufacturers[_tokenIdMergeTo]
        );
    }

    /** @dev 
        The `_tokenContractAddress` MUST be a ERCxxx contract
        The `_tokenIdMergeTo` argument MUST be the token that the manufacturers are merged into
        The `_tokenIdMergeSource` MUST be the token that the manufacturers are sourced from
    */
    function mergeExternalToken(
        address _tokenContractAddress,
        uint256 _tokenIdMergeTo,
        uint256 _tokenIdMergeSource
    ) public _isManufacturer(_tokenIdMergeTo) {
        _authorizedForToken(_msgSender(), _tokenIdMergeTo);

        RecallToken tokenContract = RecallToken(_tokenContractAddress);

        for (
            uint256 i = 0;
            i <
            tokenContract.getManufacturersOfToken(_tokenIdMergeSource).length;
            i++
        ) {
            appendUniqueManufacturerOfToken(
                tokenContract.getManufacturersOfToken(_tokenIdMergeSource)[i],
                _tokenIdMergeTo
            );
        }

        emit TokenMerged(
            _msgSender(),
            _tokenIdMergeTo,
            _tokenIdMergeSource,
            manufacturers[_tokenIdMergeTo]
        );
    }

    function appendUniqueManufacturerOfToken(
        address _manufacturer,
        uint256 _tokenId
    ) private {
        bool found = false;
        for (uint256 i = 0; i < manufacturers[_tokenId].length; i++) {
            if (manufacturers[_tokenId][i] == _manufacturer) {
                found = true;
                break;
            }
        }
        if (!found) {
            manufacturers[_tokenId].push(_manufacturer);
        }
    }
}

```

## Security Considerations

Needs discussion.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
