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
    event TokenChecked(address indexed _announcer, uint256 _tokenId, TokenCheckingState _resultState);
    
    enum TokenState {OK, ON_HOLD, NOT_OK}

    enum TokenCheckingState {NONE, PLEASE_CHECK, CHECKED_NO_DEFECT, CHECKED_DEFECT}

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
    function checkToken(uint256 _tokenId, TokenCheckingState _tokenCheckingState) external;

    /**
        @notice Changes the `TokenState` for all token specified by `_tokenIds` to `{NOT_OK}`
        @dev Caller must be approved to manage the tokens
        MUST revert on any other error.
        MUST emit the `ForwardRecall` event to reflect the TokenState change     
        @param _tokenIds           The defect Token
    */
    function forwardRecall(uint256[] calldata _tokenIds) external;
```

## Backwards Compatibility

No backward compatibility issues found.

## Reference Implementation

```
pragma solidity ^0.8.19;

/**
    @title ERC-ERC4242 Recall Tracing Token
    @dev See https://eips.ethereum.org/EIPS/eip-ERC4242
    @notice This contract represents a recall tracing token standard that allows manufacturers, distributors, and customers
            to announce defects, check tokens, and forward recalls based on unique product IDs (TokenIDs). 
            It enables efficient traceability and communication within a product supply chain.
 */

contract ERC4242 is erc1155i, erc165i {
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
    event TokenChecked(address indexed _announcer, uint256 _tokenId, TokenCheckingState _resultState);
    
    enum TokenState {OK, ON_HOLD, NOT_OK}

    enum TokenCheckingState {NONE, PLEASE_CHECK, CHECKED_NO_DEFECT, CHECKED_DEFECT}

    address[] manufacturers;

    mapping(address => mapping(uint256 => TokenCheckingState)) manufacturerTokenCheckingStates;

    mapping(uint256 => TokenCheckingState) public tokenCheckingStates;

    mapping(uint256 => TokenState) public tokenStates;

    mapping(uint256 => bool) public inProduction;

    modifier _isManufacturer {
        bool tempIsManufacturer = false;
        for (uint i = 0; i < manufacturers.length; i++) {
            if (msg.sender == manufacturers[i]) {
                tempIsManufacturer = true;
                break;
            }
        }

        require(tempIsManufacturer == true, "Must be Manufacturer");
        _;
    }

    /**
        @notice Changes the `TokenState` for a token specified by `_tokenId` to `{NOT_OK}`
        @dev Caller must be approved to manage the token
        MUST revert if TokenState of `_tokenId` is {ON_HOLD, NOT_OK}.
        MUST revert on any other error.
        MUST emit the `DefectAnnounced` event to reflect the TokenState change     
        @param _tokenId   The defect Token
    */
    function announceDefect(uint256 _tokenId) public {
        emit DefectAnnounced(msg.sender, _tokenId);
        bool isManufacturer = false;
        uint selectedManufacturer = 0;
        for (uint i = 0; i < manufacturers.length; i++) {
            if (msg.sender == manufacturers[i]) {
                isManufacturer = true;
                selectedManufacturer = i;
                break;
            }
        }
        if (isManufacturer) {
            tokenStates[_tokenId] = TokenState.NOT_OK;
            return;
        } else {
            tokenStates[_tokenId] = TokenState.ON_HOLD;
        }

        for (uint i = 0; i < manufacturers.length; i++) {
            manufacturerTokenCheckingStates[manufacturers[i]][_tokenId] = TokenCheckingState.PLEASE_CHECK;
        } 
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
    function checkToken(uint256 _tokenId, TokenCheckingState _tokenCheckingState) _isManufacturer public {
        uint selectedManufacturer = 0;
        for (uint i = 0; i < manufacturers.length; i++) {
            if (msg.sender == manufacturers[i]) {
                selectedManufacturer = i;
                break;
            }
        }
        manufacturerTokenCheckingStates[msg.sender][_tokenId] = _tokenCheckingState;
    }

    /**
        @notice Changes the `TokenState` for all token specified by `_tokenIds` to `{NOT_OK}`
        @dev Caller must be approved to manage the tokens
        MUST revert on any other error.
        MUST emit the `ForwardRecall` event to reflect the TokenState change     
        @param _tokenIds           The defect Token
    */
    function forwardRecall(uint256[] calldata _tokenIds) public {
        emit ForwardRecall(msg.sender, _tokenIds);
        for (uint i = 0; i < _tokenIds.length; i ++) {
            tokenStates[_tokenIds[i]] = TokenState.NOT_OK;
        }
    }

    function transfer(address _receiver, uint256 _tokenId, bool _internal) public {
        if (inProduction[_tokenId] && _internal) {
            manufacturers.push(_receiver);
        }
        if (!_internal && !inProduction[_tokenId]) {
            inProduction[_tokenId] = false;
        }
        // safeTransferFrom()
    }
}
```

## Security Considerations

Needs discussion.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
